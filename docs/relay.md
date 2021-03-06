---
id: relay
title: Relay all-in
sidebar_label: Relay all-in
---

This file describes experimental and more advanced Relay features. It can be very unstable due to its nature so be careful. _Here be dragons!_

TODO:

- Mock Data Generation: https://github.com/facebook/relay/commit/09d317943f6936ffb0002154c389b6d7a507c58d
- `renderPolicy`: https://github.com/facebook/relay/commit/b1cf05de8770122b30d491c4265df01e161e67c9 (partial/full)
- New GC release buffer: https://github.com/mrtnzlml/relay/pull/126/commits/6ed264413ba8cdd586d695e5ed234951ee9eca13
- [complex arguments with nested variables are now supported](https://github.com/facebook/relay/commit/5da3be070283c6dcd42774ba33c1590db65fe3c7)
- HTTP persister example: https://github.com/facebook/relay/commit/aaa9588e081d3591ad8d043e924cacfadc06ec80
- TODO: special `__id` field

> There are different tradeoffs across completeness, consistency, and performance, and there isn't one approach that is appropriate for every app. Relay focuses on cases where consistency matters: if you don't need consistency then a simpler/lighter solution can be more appropriate. ([source](https://github.com/facebook/relay/issues/2237#issuecomment-525420993))

## Useful Links (learning resources):

- https://github.com/adeira/relay-example
- https://github.com/sibelius/relay-modern-network-deep-dive
- https://medium.com/entria/wrangling-the-client-store-with-the-relay-modern-updater-function-5c32149a71ac
- https://twitter.com/sseraphini/status/1078595758801203202
- https://relay-modern-course.now.sh/packages/
- https://github.com/zth/relay-utils
- https://github.com/zth/reason-relay

## Relay Config (`relay.config.js`)

Relay supports configuration via CLI but also via configuration files using official NPM package [`relay-config`](https://www.npmjs.com/package/relay-config). Configuration files work only when you install this package. Relay Config relies on [cosmiconfig](https://github.com/davidtheclark/cosmiconfig) to do its bidding. It’s configured to load from:

- a `relay` key in `package.json`

```json
{
  "relay": {
    "src": "./src"
  }
}
```

- a `relay.config.json` file

```json
{
  "src": "./src"
}
```

- or a `relay.config.js` file

```js
module.exports = {
  src: './src',
};
```

It accepts all the same configuration as the CLI does. Additionally, when using the `relay.config.js` file, a configuration entry like the language plugin also accepts an actual function:

```js
const typescript = require('relay-compiler-language-typescript');

module.exports = {
  language: typescript,
};
```

In the future, other entries such as `persistedQueries` and `customScalars` could also be configured as such and allow for projec specific setup.

See: https://github.com/facebook/relay/commit/d3ec68ec137f7d72598a6f28025e94fba280e86e

## GraphQL types without ID field

Ever wondered how are GraphQL types being stored inside Relay Store when the types doesn't have globally unique `ID!` according to GraphQL specification? Here is an example of 2 identical stores _with_ and _without_ the ID: https://gist.github.com/mrtnzlml/e77315a6879ce8de26fe2a164872be09

Basically, Relay will try to use the `ID` field when available (preferable). However, when it's not available, it will construct some unique key which represents the record correctly. Here is an example of the record _with_ ID:

```json
{
  "QWxsSG90ZWxBdmFpbGFiaWxpdHlIb3RlbDo0NTA5Njk1": {
    "__id": "QWxsSG90ZWxBdmFpbGFiaWxpdHlIb3RlbDo0NTA5Njk1",
    "__typename": "AllHotelAvailabilityHotel",
    "id": "QWxsSG90ZWxBdmFpbGFiaWxpdHlIb3RlbDo0NTA5Njk1",
    "name": "Sweet Inn Apartments - Rocafort"
  }
}
```

And here is it _without_ the `ID`:

```json
{
  "client:root:allAvailableBookingComHotels(search:{\"checkin\":\"2020-02-13\",\"checkout\":\"2020-02-15\",\"cityId\":\"SG90ZWxDaXR5Oi0zNzI0OTA=\",\"roomsConfiguration\":[{\"adultsCount\":2}]}):edges:0:node": {
    "__id": "client:root:allAvailableBookingComHotels(search:{\"checkin\":\"2020-02-13\",\"checkout\":\"2020-02-15\",\"cityId\":\"SG90ZWxDaXR5Oi0zNzI0OTA=\",\"roomsConfiguration\":[{\"adultsCount\":2}]}):edges:0:node",
    "__typename": "AllHotelAvailabilityHotel",
    "name": "Sweet Inn Apartments - Rocafort"
  }
}
```

As you can see, the ID is composed of the query itself + the path. Moreover, there are also GraphQL arguments which ensures you will always get the correct record (forementioned Relay consistency).

## Future of `QueryRenderer`/`useQuery` pattern

> In general we're planning to move away from the QueryRenderer/useQuery pattern, which we're referring to as "fetch-on-render". This design makes behavior unpredictable (rendering can happen arbitrarily due to changes in parent components, suspense can cause re-renders and doesn't guarantee cleanup). The alternative is "fetch-then-render" - perform your data-fetching based on some event (user interaction, navigation, timer, app initialization) and then consume that result during render. Then "how do i refetch?" has the same answer as "how do i fetch?". Expect to see more API changes in this direction.

Source: https://github.com/facebook/relay/issues/2864#issuecomment-535108266

## Deferred results

- https://github.com/graphql/graphql-js/pull/2318
- https://github.com/graphql/graphql-js/pull/2319
- https://gist.github.com/robrichard/f563fd272f65bdbe8742764f1a149b2b

`@defer` directive is not really ready in GraphQL world (no matter what framework) but there is a different solution which you can use today. All you need is a refetch container and `@inline` directive. Let's say you want to fetch "note" lazily for some reason. Simply wrap the component into `createRefetchContainer` instead of `createFragmentContainer` and fetch some parts of the fragment conditionally like so:

```js
export const NoteContainer = createRefetchContainer(
  NoteContainerWithoutData,
  {
    lead: graphql`
      fragment NoteContainer_lead on Lead
        @argumentDefinitions(isMounted: { type: "Boolean!", defaultValue: false }) {
        id
        ... @include(if: $isMounted) {
          ...NoteEditor_lead
          note
        }
      }
    `,
  },
  graphql`
    query NoteContainerDeferredQuery($isMounted: Boolean!, $id: ID!) {
      lead: node(id: $id) {
        ...NoteContainer_lead @arguments(isMounted: $isMounted)
      }
    }
  `,
);
```

Now, the only thing you have to do is to send the refetch query on mount and you are done:

```js
useEffect(() => {
  // you can find `relay` in your props
  relay.refetch(
    {
      isMounted: true,
      id: lead.id,
    },
    undefined,
    undefined,
    {
      fetchPolicy: 'store-or-network', // handy but not necessary
    },
  );
}, [lead.id, relay]);
```

Kudos: https://relay-modern-course.now.sh/packages/11-simulating-defer/#2

## Custom Relay Compiler

Most of the people are OK with the default OSS version of Relay Compiler. However, it can be sometimes beneficial to write your own Relay Compiler in order to achieve some advanced features (custom behavior or persisting queries to your server for example). Facebook also uses internally their own Relay Compiler implementation. Here is one example of "why" (source: https://github.com/facebook/relay/commit/f1e2e79462d593d73efb80727bc5dd56b1c43cf6#commitcomment-36337550).

The default config generates the flow types inline in the generated files, so something like:

```text
meeting: {
  response: 'GOING' | 'NOT_GOING'
}
```

This can in some cases introduce a bunch of noise if the generated files are checked in and the schema changes frequently. For that purpose, we instead generate something like:

```text
import type {MeetingResponse} from 'MeetingResponse.enums';
meeting: {
  response: MeetingResponse
}
```

Doing that in OSS as well would increase the number of generated files and also add the question of where to put these files and how to import them.

## RelayResponseNormalizer: `handleStrippedNulls`

Please read this: https://github.com/facebook/relay/issues/3052

Relay is able to recover completely missing fields in the response. You can use this knowledge to optimize JSON response from the server. Let's say this is our incoming payload from the server:

```json
{
  "data": {
    "allLocations": {
      "edges": [
        { "node": { "id": "san-francisco_ca_us", "name": "San Francisco" } },
        { "node": { "id": "boston_ma_us", "name": "Boston" } },
        { "node": { "id": "washington_dc_us", "name": "Washington, D.C." } }
      ]
    }
  }
}
```

Traditionally, server would return something like this in case of failure (or just missing data):

```json
{
  "data": {
    "allLocations": {
      "edges": [
        { "node": { "id": "san-francisco_ca_us", "name": "San Francisco" } },
        { "node": { "id": "boston_ma_us", "name": null } },
        { "node": { "id": "washington_dc_us", "name": null } }
      ]
    }
  },
  "errors": ...
}
```

But it's not necessary to send the nullable fields at all. Afterall, server knows what fields were requested. `RelayResponseNormalizer` by default recovers from this state so you can send response like this from the server (see the missing names):

```json
{
  "data": {
    "allLocations": {
      "edges": [
        { "node": { "id": "san-francisco_ca_us", "name": "San Francisco" } },
        { "node": { "id": "boston_ma_us" } },
        { "node": { "id": "washington_dc_us" } }
      ]
    }
  },
  "errors": ...
}
```

Relay will show you this warning in this console (dev mode only):

> Warning: RelayResponseNormalizer(): Payload did not contain a value for field `name: name`. Check that you are parsing with the same query that was used to fetch the payload.

See: https://github.com/facebook/relay/blob/76fef685f70a5aa09cd180ce0f2ef6b6d3f4f7e8/packages/relay-runtime/store/RelayResponseNormalizer.js#L75

## Refetch container

https://facebook.github.io/relay/docs/en/refetch-container.html

When `refetch` is called and the `refetchQuery` is executed, Relay doesn't actually use the result of the query to re-render the component. All it does is normalize the payload into the store and fire any relevant subscriptions. This means that if the fetched data is unrelated to the data that the mounted container is subscribed to (e.g. using a totally different node id that doesn't have any data overlaps), then the component won't re-render.

Refetch containers are only really meant to be used when you are changing variables in the component fragment. If you don't want or need to include variables in the fragment, you could go one level up and set new variables directly in the QueryRenderer (using props or state).

https://github.com/facebook/relay/issues/2244#issuecomment-355054944

## RelayNetworkLogger

TODO: https://github.com/facebook/relay/issues/2674 !

```js
import RelayNetworkLogger from 'relay-runtime/lib/RelayNetworkLogger';

import fetchFunction from './fetchFunction';
import subscribeFunction from './subscribeFunction';

const fetch = __DEV__ ? RelayNetworkLogger.wrapFetch(fetchFunction) : fetchFunction;

const subscribe = __DEV__ ? RelayNetworkLogger.wrapSubscribe(subscribeFunction) : subscribeFunction;

const network = Network.create(fetch, subscribe);
const source = new RecordSource();
const store = new Store(source);

const env = new Environment({
  network,
  store,
});

export default env;
```

## RelayObservable.onUnhandledError

You can override default behavior of unhandled errors when using Relay Observable:

```js
import { Observable } from 'relay-runtime';

if (__DEV__) {
  Observable.onUnhandledError((error, isUncaughtThrownError) => {
    console.error(error);
  });
}

Observable.create( ... )
```

Default [implementation](https://github.com/facebook/relay/blob/8f4d54522440a8146de794e72ea5bf873016b408/packages/relay-runtime/network/RelayObservable.js#L616-L636):

```js
if (__DEV__) {
  // Default implementation of HostReportErrors() in development builds.
  // Can be replaced by the host application environment.
  RelayObservable.onUnhandledError((error, isUncaughtThrownError) => {
    declare function fail(string): void;
    if (typeof fail === 'function') {
      // In test environments (Jest), fail() immediately fails the current test.
      fail(String(error));
    } else if (isUncaughtThrownError) {
      // Rethrow uncaught thrown errors on the next frame to avoid breaking
      // current logic.
      setTimeout(() => {
        throw error;
      });
    } else if (typeof console !== 'undefined') {
      // Otherwise, log the unhandled error for visibility.
      // eslint-disable-next-line no-console
      console.error('RelayObservable: Unhandled Error', error);
    }
  });
}
```

## Common Relay problems (from user perspective)

- users are not using fragments correctly (data-masking misunderstanding)
- incorrect environment imports (not using the right Environment instance)

TKTK

## Common Relay errors explained

### Relay does not allow `__typename` field on Query, Mutation or Subscription

> Rather than special-case the representation of the root of the graph, Relay generates a "client" record to represent the Query and mutation/subscription objects. Like all other record instances those records have a **typename, but we special-case this typename to be 'ROOT'. Querying for the **typename field would overwrite this value with the actual typename (e.g. Query or whatever you call it in your schema), which messes with a few invariants. It's on our wishlist to make the root record a bit less special, but in practice we couldn't think of a reason to query \_\_typename on the root so we just disallow it for now.

- https://github.com/facebook/relay/commit/793729e7af9c7ee0de971e3d2ed26e5896774640#commitcomment-37652508
