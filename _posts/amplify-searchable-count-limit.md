---
title: "The Hidden 10,000-Document Limit in AWS Amplify's @searchable Directive"
excerpt: "AWS Amplify's @searchable directive makes it trivial to bolt full-text search onto a GraphQL API—until you need an accurate document count past 10,000 records. Here's why that happens and how we worked around it."
coverImage: "/assets/posts/amplify-searchable-count-limit/cover.png"
date: "2026-07-15T13:00:00-04:00"
author: rahul
ogImage:
  url: "/assets/posts/amplify-searchable-count-limit/cover.png"
---

At Picketa, we lean on [AWS Amplify Gen 1](https://docs.amplify.aws/gen1/) for a lot of our GraphQL APIs, and the [`@searchable`](https://docs.amplify.aws/gen1/react/build-a-backend/graphqlapi/search-and-result-aggregations/) is one of its more convenient features—annotate a model, and Amplify wires up an [OpenSearch](https://opensearch.org/docs/latest/) domain and streaming pipeline for you, no infrastructure code required. It works great, until you need something as simple as an accurate total count on a dataset larger than 10,000 documents.

## The Problem

We have a `ScanAnalytics` model—one record per tissue scan event—marked [`@model`](https://docs.amplify.aws/gen1/react/build-a-backend/graphqlapi/data-modeling/) [`@searchable`](https://docs.amplify.aws/gen1/react/build-a-backend/graphqlapi/search-and-result-aggregations/), with a `cropId` field identifying which crop the scan belongs to. We wanted to show users how many scans exist for a given crop, e.g. "14,382 scans for Potato." Once the number of matching documents for a given `cropId` crosses roughly 10,000, that count silently caps out and stops moving. Add another thousand scans for `cropId`, and the total still reads exactly "10000".

This isn't an Amplify-specific number—it comes straight from the search engine underneath.

## Why This Happens

`@searchable` is backed by OpenSearch, and Amplify generates a resolver that issues a standard [`_search`](https://docs.opensearch.org/latest/api-reference/search-apis/search/) request. By default, OpenSearch's `hits.total` stops counting exactly once it hits the [`track_total_hits`](https://opensearch.org/docs/latest/api-reference/search/) threshold—10,000 by default—and reports `10000` instead of an exact number.

That number isn't arbitrary. It comes from [the original Elasticsearch change](https://github.com/elastic/elasticsearch/pull/37466) that introduced this default, and the engineer who wrote it was explicit about why: *"I choose 10,000 as the default because that's also the number we use to limit pagination."* In other words, it's tied directly to `index.max_result_window`— [OpenSearch's own default cap](https://docs.opensearch.org/latest/install-and-configure/configuring-opensearch/index-settings/) on how deep you can paginate into a result set. If you can't page past result 10,000 anyway, there's little point paying to count every match beyond it. It's a sensible trade-off for typical search UIs, just not for anything that needs an actual total.

Amplify's generated resolver doesn't expose any way to configure this—you get whatever the default request template gives you.

## The Catch: A total Field That Lies

Here's the part that confused us. [`@searchable`](https://docs.amplify.aws/gen1/react/build-a-backend/graphqlapi/search-and-result-aggregations/) doesn't just generate a search query—it generates a `Searchable<ModelName>Connection` type with a `total: Int` field already sitting right there. For our `ScanAnalytics` type, the generated connection type looks like this:

```graphql
type SearchableScanAnalyticsConnection @aws_api_key @aws_cognito_user_pools {
  items: [ScanAnalytics]!
  nextToken: String
  total: Int
  aggregateItems: [SearchableAggregateResult]!
}
```

And the generated response resolver (Amplify compiles these down to VTL under `amplify/backend/api/<api-name>/build/resolvers/`, e.g. `Query.searchScanAnalytics.res.vtl`) populates it directly from OpenSearch's own count:

```vtl
$util.toJson({
  "items": $es_items,
  "total": $ctx.result.hits.total.value,
  "nextToken": $nextToken,
  "aggregateItems": $aggregateValues
})
```

So `total` isn't missing—it's wired up end to end. The problem is one line earlier, in the *request* template (`Query.searchScanAnalytics.req.vtl`), which builds the OpenSearch request body. Here are the closing lines of that template, right where the request body object is assembled and returned:

```vtl
...
"size": #if( $args.limit ) $args.limit #else 100 #end,
"sort": $sortValues,
"version": false,
"query": $util.toJson($filter),
"aggs": $util.toJson($aggregateValues)
```

Amplify built all the plumbing for an exact count and then left the underlying request in its default, early-terminating mode. The field exists, is fully wired, and quietly lies to you once your result set gets large enough.

## The Naive Fix: track_total_hits: true

The generated `amplify/backend/api/<api-name>/build/resolvers/` files aren't meant to be hand-edited—`amplify push` regenerates them. The supported [override point](https://docs.amplify.aws/gen1/react/tools/cli-legacy/overwrite-customize-resolvers/#overwriting-resolvers) is the sibling (non-`build`) `amplify/backend/api/<api-name>/resolvers/` directory: any file you drop in there with a matching name gets deployed as-is instead of Amplify's generated version. So the direct fix is to copy the generated request template out, add one line, and place it at `amplify/backend/api/<api-name>/resolvers/Query.searchScanAnalytics.req.vtl`. Everything above stays untouched—only the closing lines of the request body change, with [`track_total_hits`](https://opensearch.org/docs/latest/api-reference/search/) inserted:

```vtl
...
"size": #if( $args.limit ) $args.limit #else 100 #end,
"sort": $sortValues,
"version": false,
"track_total_hits": true,
"query": $util.toJson($filter),
"aggs": $util.toJson($aggregateValues)
```

This works—`total` now reflects the real count no matter how large it gets. The trade-off is cost: [`track_total_hits: true`](https://opensearch.org/docs/latest/api-reference/search/) tells OpenSearch it can no longer early-terminate the count once it hits the default threshold, so it has to visit every matching document across every shard. Since `searchScanAnalytics` is also sorting and fetching full `_source` documents on every call, you're now paying that full exact-count penalty on every single page load—even when nobody looks past the first page. You're also on the hook for keeping a hand-maintained copy of Amplify's generated VTL in sync as the schema evolves.

## A Cheaper Alternative: OpenSearch's Own _count API

Even if you need the total on every page—say, a "Showing 20 of 14,382 scans" label that has to stay accurate across every paginated request—it's cheaper in aggregate to run two lean queries than one heavy one: keep the paginated search exactly as Amplify generates it (no [`track_total_hits`](https://opensearch.org/docs/latest/api-reference/search/), still early-terminating) and fire a second, purpose-built query just for counting alongside it.

And it turns out OpenSearch already ships a dedicated endpoint for exactly this: [`_count`](https://opensearch.org/docs/latest/api-reference/count/). It's a sibling to `_search` that takes the same query DSL but only ever returns a document count—no hits, no scoring, no sorting—and critically, it isn't subject to the [`track_total_hits`](https://opensearch.org/docs/latest/api-reference/search/) cap at all, since there's no search result window to early-terminate in the first place. [OpenSearch's own docs](https://opensearch.org/docs/latest/api-reference/count/) are explicit that this isn't just a narrower API for convenience—it's actually cheaper to run: *"The Count API is more efficient than using the Search API with `size: 0`... because it is optimized specifically for counting operations."* 
We implemented this as a modern [`APPSYNC_JS`](https://docs.aws.amazon.com/appsync/latest/devguide/resolver-reference-overview-js.html) Unit resolver—a single JS file with `request()`/`response()` functions, deployed through a hand-authored [nested CloudFormation stack](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-resource-appsync-resolver.html), reusing the same `OpenSearchDataSource` that `@searchable` already provisions.

Schema addition:

```graphql
type Query {
  getScanAnalyticsCount(cropId: ID!): Int
}
```

`resolvers/Query.getScanAnalyticsCount.js`:

```js
import { util } from '@aws-appsync/utils';

export function request(ctx) {
  const indexPath = '/scananalytics/_count';

  const { cropId } = ctx.args;

  if (!cropId) {
    util.error('Invalid input: invalid cropId', 'InvalidInputError');
  }

  const body = {
    query: {
      term: { 'cropId.keyword': cropId },
    },
  };

  return {
    version: '2018-05-29',
    operation: 'GET',
    path: indexPath,
    params: { body },
  };
}

export function response(ctx) {
  const { count } = ctx.result;
  return count;
}
```

Calling `getScanAnalyticsCount(cropId: "potatoId")` returns the exact number of scans for that crop, no matter how far past 10,000 it goes—and there's no `size`, [`track_total_hits`](https://opensearch.org/docs/latest/api-reference/search/), or sort to think about at all. 
The resolver follows a few simple conventions:
- argument validation via `util.error(message, errorType)` before the request is built, 
- and a raw HTTP passthrough shape (`version` / `operation` / `path` / `params.body`)—the exact request format [AppSync's OpenSearch/Elasticsearch JS resolver reference](https://docs.aws.amazon.com/appsync/latest/devguide/resolver-reference-elasticsearch-js.html) documents—rather than any higher-level query builder.

Wiring it up uses that same [nested-stack](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-resource-appsync-resolver.html) mechanism, added as a new resource in `amplify/backend/api/<api-name>/stacks/CustomResources.json`, referencing the existing `OpenSearchDataSource`:

```json
"GetScanAnalyticsCountResolver": {
  "Type": "AWS::AppSync::Resolver",
  "Properties": {
    "ApiId": { "Ref": "AppSyncApiId" },
    "DataSourceName": "OpenSearchDataSource",
    "TypeName": "Query",
    "FieldName": "getScanAnalyticsCount",
    "Runtime": { "Name": "APPSYNC_JS", "RuntimeVersion": "1.0.0" },
    "CodeS3Location": {
      "Fn::Sub": [
        "s3://${S3DeploymentBucket}/${S3DeploymentRootKey}/resolvers/Query.getScanAnalyticsCount.js",
        { "S3DeploymentBucket": { "Ref": "S3DeploymentBucket" }, "S3DeploymentRootKey": { "Ref": "S3DeploymentRootKey" } }
      ]
    }
  }
}
```

`amplify push` uploads the JS file to S3 and deploys the resolver alongside everything else—AppSync doesn't distinguish between a "generated" resolver and a "custom" one once they're both attached to the API. 

## Trade-offs to Keep in Mind

- **Two round trips instead of one.** Even though each query is individually cheaper, you're now managing two requests instead of one—an extra network hop, and a second resolver to deploy and maintain alongside the generated one.
- **Still O(matching documents).** For queries that match a huge fraction of your index, both approaches are inherently expensive. There's no shortcut around exact counting in [OpenSearch](https://opensearch.org/docs/latest/)/Elasticsearch; you're only avoiding *redundant* work, not the counting cost itself.
- **Customizing generated resolvers means owning them.** Whether you fork the VTL or add a standalone JS resolver, you're responsible for keeping filters and permissions in sync as the schema evolves—Amplify won't do it for you.

## Conclusion

The 10,000-document count cap isn't a bug—it's [OpenSearch](https://opensearch.org/docs/latest/)'s default behavior leaking through an abstraction that doesn't give you any knobs to control it, even though Amplify already built a `total` field that implies it should just work. Customizing the resolver layer is the only way around it today, and we found it's worth choosing a dedicated, standalone count resolver over forking the generated VTL: it's cheaper to run, and it doesn't put you in the business of hand-maintaining Amplify's generated templates.

This is just the first of a few Amplify rough edges we've run into. Next up: `@searchable`'s `aggregates` support passes straight through to OpenSearch's `terms` aggregation, which defaults to returning only 10 buckets—we'll cover how we made that configurable so callers can ask for their actual top K. Stay tuned.