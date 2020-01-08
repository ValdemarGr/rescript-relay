---
id: pagination
title: Pagination
sidebar_label: Pagination
---

_This section is a WIP_

#### Recommended background reading

- [Queries and mutations in GraphQL](https://graphql.org/learn/queries/)
- [A Guided Tour of Relay: Rendering List Data and Pagination](https://relay.dev/docs/en/experimental/a-guided-tour-of-relay#rendering-list-data-and-pagination)
- [The Relay server specification: Connections](https://relay.dev/docs/en/graphql-server-specification.html#connections)
- [React documentation: Suspense for Data Fetching](https://reactjs.org/docs/concurrent-mode-suspense.html)

## Pagination in Relay

> The features outlined on this page requires that your schema follow the [Relay specification](https://relay.dev/docs/en/experimental/graphql-server-specification.html). Read more about using ReasonRelay with schemas that don't conform to the Relay specification [here](using-with-schemas-that-dont-conform-to-the-relay-spec).

Relay has some great built-in tools to make pagination very simple if you use [connection-based pagination](https://relay.dev/docs/en/graphql-server-specification.html#connections) and your schema conforms to the [the Relay server specification](https://relay.dev/docs/en/graphql-server-specification.html). Let's look at some examples.

### Setting up for pagination

Pagination is always done using a [fragment](using-fragments). Here's a definition of a Relay [fragment](using-fragments) that'll allow you to paginate over the connection `ticketsConnection`:

```reason
module Fragment = [%relay.fragment
  {|
  fragment RecentTickets_query on Query
    @refetchable(queryName: "RecentTicketsRefetchQuery")
    @argumentDefinitions(
      count: {type: "Int!", defaultValue: 10},
      cursor: {type: "String!", defaultValue: ""}
    ) {
    ticketsConnection(first: $count, after: $cursor)
      @connection(key: "RecentTickets_ticketsConnection")
    {
      edges {
        node {
          id
          ...SingleTicket_ticket
        }
      }
    }
  }
|}
];
```

Quite a few directives and annotations used here. Let's break down what's going on:

1. First off, this particular fragment is defined on the `Query` root type (the root query type is really just like any other GraphQL type). This is just because the `ticketsConnection` field happen to be on `Query`, pagination can be done on fields on any GraphQL type.
2. We make our fragment _refetchable_ by adding the `@refetchable` directive to it. You're encouraged to read [refetching and loading more data](refetching-and-loading-more-data) for more information on making fragments refetchable.
3. We add another directive, `@argumentDefinitions`, where we define two arguments that we need for pagination, `count` and `cursor`. This component is responsible for paginating itself, so we want anyone to be able to use this fragment without providing those arguments for the initial render. To solve that we add default values to our arguments. [You're encouraged to read more about `@argumentDefinitions` here](https://relay.dev/docs/en/experimental/a-guided-tour-of-relay#arguments-and-argumentdefinitions).
4. We select the `ticketsConnection` field on `Query` and pass it our pagination arguments. We also add a `@connection` directive to the field. This is important, because it tells Relay that we want it to help us paginate this particular field. By annotating with `@connection` and passing a `keyName`, Relay will understand how to find and use the field for pagination. This in turn means we'll get access to a bunch of hooks and functions for paginating and dealing with the pagination in the store. You can read more about `@connection` [here](https://relay.dev/docs/en/experimental/a-guided-tour-of-relay#adding-and-removing-items-from-a-connection).
5. Finally, we spread another component's fragment `SingleTicket_ticket` on the connection's `node`, since that's the component we'll use to display each ticket.

We've now added everything we need to enable pagination for this fragment.

## Pagination in a component

Let's look at a component that uses the fragment above for pagination:

```reason
[@react.component]
let make = (~query as queryRef) => {
  let ReasonRelay.{data, hasNext, isLoadingNext, loadNext} =
    Fragment.usePagination(queryRef);

  <div className="card">
    <div className="card-body">
      <h4 className="card-title"> {React.string("Recent Tickets")} </h4>
      <div>
        {data##ticketsConnection
         |> ReasonRelayUtils.collectConnectionNodes
         |> Array.map(ticket => <SingleTicket key=ticket##id ticket />)
         |> React.array}
        {hasNext
           ? <button
               onClick={_ => loadNext(~count=2, ~onComplete=None) |> ignore}
               disabled=isLoadingNext>
               {React.string(isLoadingNext ? "Loading..." : "More")}
             </button>
           : React.null}
      </div>
    </div>
  </div>;
};
```

Whew, plenty more to break down:

1. Just like with anything [using fragments](using-fragments), we'll need a fragment reference to pass to our fragment.
2. We pass our fragment reference into `Fragment.usePagination`. This gives us a record back containing functions and props that'll help us with our pagination. Notice `ReasonRelay.{...}` - that's a _hint_ to tell the compiler where to find the record we're using, which in turn is because the compiler cannot infer what record we're trying to destructure from, so we need to give it a hint. Check out the [full API reference here](#api-reference).
3. We gather up all the connection nodes we have by using the `ReasonRelayUtils.collectConnectionNodes` helper, which collects all available nodes on a connection to a normal array for you. Read more about [utilites provided by ReasonRelay here](utilities).
4. We render each node using the `<SingleTicket />` component, who's data demands we spread on the `node` of our refetchable fragment.
5. Finally, we use the helpers provided by `usePagination` to render a _Load more-button_ if there's more data to load, and disable it if a request is already in flight.

There, basic pagination! Relay really does all the heavy lifting for us here, which is great. Continue reading for some advanced pagination concepts and a full API reference, or move on to [subscriptions](subscriptions).

## Two types of pagination

Relay provides two types of pagination by default;

1. _"Normal"_ pagination, which we saw above using `usePagination`. This type of pagination is _not integrated with suspense_, and will give you two flags `isLoadingNext` and `isLoadingPrevious` to indicate whether a request is in flight.
2. _Blocking_ pagination, which is done via `useBlockingPagination` and _is integrated with suspense_. This is more suitable for a "load all"-type of pagination.

You're encouraged to read more in the [official Relay documentation on pagination](https://relay.dev/docs/en/experimental/a-guided-tour-of-relay#pagination) for more information on the two types of pagination.

## API Reference

A `[%relay.fragment]` which is annotated with a `@refetchable` directive, and which contains a `@connection` directive somewhere, has the following functions added to it's module, in addition to everything mentioned in [using fragments](using-fragments):

### `usePagination`

As shown above, `usePagination` provides helpers for paginating your fragment/connection.

> `usePagination` uses Relay's `usePaginationFragment` under the hood, which you can [read more about here](https://relay.dev/docs/en/experimental/api-reference#usepaginationfragment).

##### Parameters

`usePagination` returns a record with the following properties.

| Name                | Type                                                                                                      | Note                                                                 |
| ------------------- | --------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------- |
| `data`              | `'fragmentData`                                                                                           | The data as defined by the fragment.                                 |
| `loadNext`          | `(~count: int, ~onComplete: option(option(Js.Exn.t) => unit)) => Disposable.t;`                           | A function for loading the next `count` nodes of the connection.     |
| `loadPrevious`      | `(~count: int, ~onComplete: option(option(Js.Exn.t) => unit)) => Disposable.t;`                           | A function for loading the previous `count` nodes of the connection. |
| `hasNext`           | `bool`                                                                                                    | Are there more nodes forward in the connection to fetch?             |
| `hasPrevious`       | `bool`                                                                                                    | Are there more nodes backwards in the connection to fetch?           |
| `isLoadingNext`     | `bool`                                                                                                    |                                                                      |
| `isLoadingPrevious` | `bool`                                                                                                    |                                                                      |
| `refetch`           | `(~variables: 'variables, ~fetchPolicy: fetchPolicy=?, ~onComplete: option(Js.Exn.t) => unit=?, unit) =>` | Refetch the entire connection with potentially new variables.        |

### `useBlockingPagination`

Integrated with _suspense_, meaning it will suspend the component if used.

> `useBlockingPagination` uses Relay's `useBlockingPaginationFragment` under the hood, which you can [read more about here](https://relay.dev/docs/en/experimental/api-reference#useblockingpaginationfragment).

##### Parameters

`useBlockingPagination` returns a record with the same properties as [usePagination](#usepagination), _excluding_ `isLoadingNext` and `isLoadingPrevious`, as that is handled by suspense.