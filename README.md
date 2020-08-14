# reason-apollo-hooks

<!-- ALL-CONTRIBUTORS-BADGE:START - Do not remove or modify this section -->

[![All Contributors](https://img.shields.io/badge/all_contributors-18-orange.svg?style=flat-square)](#contributors-)

<!-- ALL-CONTRIBUTORS-BADGE:END -->

Reason bindings for the official [@apollo/react-hooks](https://www.npmjs.com/package/@apollo/react-hooks)

## Table of contents

- [reason-apollo-hooks](#reason-apollo-hooks)
  - [Table of contents](#table-of-contents)
  - [Installation :arrow_up:](#installation-arrow_up)
  - [Setting up :arrow_up:](#setting-up-arrow_up)
  - [Available hooks :arrow_up:](#available-hooks-arrow_up)
    - [useQuery :arrow_up:](#usequery-arrow_up)
    - [useMutation :arrow_up:](#usemutation-arrow_up)
    - [useSubscription :arrow_up:](#usesubscription-arrow_up)
  - [Cache :arrow_up:](#cache-arrow_up)
  - [Getting it running](#getting-it-running)
  - [Contributors ✨](#contributors-)

## Installation [:arrow_up:](#table-of-contents)

```sh
yarn add reason-apollo-hooks @apollo/react-hooks @apollo/link-context @apollo/link-error @apollo-upload-client graphql-tag
```

Follow the installation instructions of [@reasonml-community/graphql-ppx](https://github.com/reasonml-community/graphql-ppx).

Then update your bsconfig.json

```diff
"bs-dependencies": [
  ...
+ "reason-apollo-hooks"
]
```

## Setting up [:arrow_up:](#table-of-contents)

Add the provider in the top of the tree

```reason
/* Create an InMemoryCache */
let inMemoryCache = ApolloClient.InMemoryCache.createInMemoryCache();

/* Create an HTTP Link */
let httpLink =
  ApolloClient.Link.HttpLink.createHttpLink(~uri="http://localhost:3010/graphql", ());

let client =
  ApolloClient.Client.createApolloClientJS({
    link: httpLink,
    cache: inMemoryCache
  });

let app =
  <ApolloClient.Provider client>
    ...
  </ApolloClient.Provider>;
```

## Available hooks [:arrow_up:](#table-of-contents)

### useQuery [:arrow_up:](#table-of-contents)

```reason
open ApolloClient;

module UserQuery = [%graphql {|
  query UserQuery {
    currentUser {
      name
    }
  }
|}];

[@react.component]
let make = () => {
  /* Both variant and records available */
  let result = useQuery(~variables=(), (module UserQuery));

  <div>
  {
    switch(
      result.loading,
      result.error,
      result.data
    ) {
    | (true, _, _) => <p>{React.string("Loading...")}</p>
    | (false, _, Some(data)) =>
      <p>{React.string(data.currentUser.name)}</p>
    | (false, _, None)
    | (false, Some(_), _) => <p>{React.string("Get off my lawn!")}</p>
    }
  }
  </div>
}
```

Using `fetchPolicy` to change interactions with the `apollo` cache, see [apollo docs](https://www.apollographql.com/docs/react/api/react-apollo/#optionsfetchpolicy).

```reason
let result = useQuery(~fetchPolicy=NetworkOnly, (module UserQuery));
```

Using `errorPolicy` to change how errors are handled, see [apollo docs](https://www.apollographql.com/docs/react/api/react-apollo/#optionserrorpolicy).

```reason
let result = useQuery(~errorPolicy=All, (module UserQuery));
```

Using `skip` to skip query entirely, see [apollo docs](https://www.apollographql.com/docs/react/api/react-apollo/#configskip).

```reason
let (simple, _full) =
  useQuery(
    ~skip=
      switch (value) {
      | None => true
      | _ => false
      },
    (module UserQuery),
  );
```

### useMutation [:arrow_up:](#table-of-contents)

```reason
open ApolloClient;

module ScreamMutation = [%graphql {|
  mutation ScreamMutation($screamLevel: Int!) {
    scream(level: $screamLevel) {
      error
    }
  }
|}];

[@react.component]
let make = () => {
  /* Both variant and records available */
  let (screamMutation, result) =
    useMutationWithVariables(
      ~variables=ScreamMutation.makeVariables(~screamLevel=10, ()),
      (module ScreamMutation),
    );

  let scream = _ => {
    screamMutation();

    switch(result.loading, result.error, result.data) {
    | (false, _, Some(data)) => ...
    | (false, Some(error), _) => ...
    | (false, _, None) => ...
    | (true, _, _) => ...
    ;
  };

  <div>
    <button onClick={scream}>
      {React.string("You kids get off my lawn!")}
    </button>
  </div>
}
```

If you don't know the value of the variables yet you can pass them in later

```reason
open ApolloClient;

[@react.component]
let make = () => {
  /* Both variant and records available */
  let (screamMutation, result) = useMutation((module ScreamMutation));

  let scream = (_) => {
    screamMutation(ScreamMutation.makeVariables(
      ~screamLevel=10, ()
    ))

    switch(result.loading, result.error, result.data) {
    | (false, _, Some(data)) => ...
    | (false, Some(error), _) => ...
    | (false, _, None) => ...
    | (true, _, _) => ...
    ;
  };

  <div>
    <button onClick={scream}>
      {React.string("You kids get off my lawn!")}
    </button>
  </div>
}
```

### useSubscription [:arrow_up:](#table-of-contents)

In order to use subscriptions, you first need to set up your websocket link:

```diff
/* Create an InMemoryCache */
let inMemoryCache = ApolloInMemoryCache.createInMemoryCache();

/* Create an HTTP Link */
let httpLink =
  ApolloLinks.createHttpLink(~uri="http://localhost:3010/graphql", ());
+
+/* Create a WS Link */
+let webSocketLink =
+  ApolloClient.Link.webSocketLink({
+    uri: "wss://localhost:3010/graphql",
+    options: {
+      reconnect: true,
+      connectionParams: None,
+    },
+  });
+
+/* Using the ability to split links, you can send data to each link
+   depending on what kind of operation is being sent */
+let link =
+  ApolloClient.Link.split(
+    operation => {
+      let operationDefition =
+        ApolloUtilities.getMainDefinition(operation.query);
+      operationDefition.kind == "OperationDefinition"
+      && operationDefition.operation == "subscription";
+    },
+    webSocketLink,
+    httpLink,
+  );

let client =
-  ApolloClient.createApolloClient(~link=httpLink, ~cache=inMemoryCache, ());
+  ApolloClient.createApolloClient(~link, ~cache=inMemoryCache, ());

let app =
 <ApolloClient.Provider client>
   ...
 </ApolloClient.Provider>
```

Then, you can implement `useSubscription` in a similar manner to `useQuery`

```reason
open ApolloClient;

module UserAdded = [%graphql {|
  subscription userAdded {
    userAdded {
      id
      name
    }
  }
|}];

[@react.component]
let make = () => {
  let result = useSubscription((module UserAdded));

  switch (result.loading, result.error, result.data) {
    | (true, _, _) => <div> {ReasonReact.string("Loading")} </div>
    | (false, Some(error), _) =>
      <div> {ReasonReact.string(error.message)} </div>
    | (false, _, None) => ReasonReact.null
    | (false, _, Some(_response)) =>
      <audio autoPlay=true>
      <source src="notification.ogg" type_="audio/ogg" />
      <source src="notification.mp3" type_="audio/mpeg" />
    </audio>
  };
};
```

## Cache [:arrow_up:](#table-of-contents)

There are a couple of caveats with manual cache updates.

1. If you need to remove items from cached data, it is enough to just filter them out and save the result into cache as is.
2. If you need to add the result of a mutation to a list of items with the same shape, you simply concat it with the list and save into cache as it.
3. When you need to update a field, you have to resort to raw javascript to use spread operator on `Js.t` object in order to preserve `__typename` that `apollo` adds to all queries by default.

An example of cache update could look like this:

```reason
module PersonsQuery = [%graphql
{|
  query getAllPersons  {
    allPersons  {
      id
      age
      name
    }
  }
|}
];

module PersonsReadQuery = ApolloClient.ReadQuery(PersonsQuery);
module PersonsWriteQuery = ApolloClient.WriteQuery(PersonsQuery);

external cast: Js.Json.t => PersonsQuery.t = "%identity";

let updatePersons = (~client, ~name, ~age) => {
  let query = PersonsQuery.make();
  let readQueryOptions = ApolloClient.Utils.toReadQueryOptions(query);

  // can throw exception of cache is empty
  switch (PersonsReadQuery.readQuery(client, readQueryOptions)) {
  | exception _ => ()
  | cachedResponse =>
    switch (cachedResponse |> Js.Nullable.toOption) {
    | None => ()
    | Some(cachedPersons) =>
      // readQuery will return unparsed data with __typename field, need to cast it since
      // it has type Json.t, but we know it will have the same type as PersonsReadQuery.t
      let persons = cast(cachedPersons);

      // to remove items, simply filter them out
      let updatedPersons = {
        "allPersons":
          Belt.Array.keep(persons.allPersons, person => person.age !== age),
      };

      // when updating items, __typename must be preserved, but since it is not a record,
      // can't use spread, so use JS to update items
      let updatedPersons = {
        "allPersons":
          Belt.Array.map(persons.allPersons, person =>
            person.name === name ? [%bs.raw {| {...person, age } |}] : person
          ),
      };

      PersonsWriteQuery.make(
        ~client,
        ~variables=query##variables,
        ~data=updatedPersons,
        (),
      );
    }
  };
};
```

`reason-apollo-hooks` parses response data from a query or mutation using parse function created by `graphql_ppx`. For example, when using `@bsRecord` directive, the response object will be parsed from a `Js.t` object to a reason record. In this case, the response data in reason code is not the same object that is stored in cache, since `react-apollo` saves data in cache before it is parsed and returned to the component. However, when updating cache, the data must be in the same format or apollo cache won't work correctly and throw errors.

If using directives like `@bsRecord`, `@bsDecoder` or `@bsVariant` in `graphql_ppx`, the data needs to be serialized back to JS object before it is written in cache. Since there is currently no way to serialize this data (see [this issue](https://github.com/mhallin/graphql_ppx/issues/71) in `graphql_ppx`), queries that will be updated in cache shouldn't use any of those directive, unless you will take care of the serialization yourself.

By default, apollo will add field `__typename` to the queries and will use it to normalize data and manipulate cache (see [normalization](https://www.apollographql.com/docs/react/advanced/caching/#normalization)). This field won't exist on parsed reason objects, since it is not included in the actual query you write, but is added by apollo before sending the query. Since `__typename` is crucial for the cache to work correctly, we need to read data from cache in its raw unparsed format, which is achieved with `readQuery` from `ApolloClient.ReadQuery` defined in `reason-apollo`.

## Getting it running

```sh
yarn
yarn start
```

## Contributors ✨

Thanks goes to these wonderful people ([emoji key](https://allcontributors.org/docs/en/emoji-key)):

<!-- ALL-CONTRIBUTORS-LIST:START - Do not remove or modify this section -->
<!-- prettier-ignore-start -->
<!-- markdownlint-disable -->
<table>
  <tr>
    <td align="center"><a href="http://twitter.com/fakenickels"><img src="https://avatars0.githubusercontent.com/u/1283200?v=4" width="100px;" alt=""/><br /><sub><b>Gabriel Rubens</b></sub></a><br /><a href="https://github.com/Astrocoders/reason-apollo-hooks/commits?author=fakenickels" title="Code">💻</a> <a href="https://github.com/Astrocoders/reason-apollo-hooks/commits?author=fakenickels" title="Documentation">📖</a> <a href="#ideas-fakenickels" title="Ideas, Planning, & Feedback">🤔</a></td>
    <td align="center"><a href="https://github.com/arielschiavoni"><img src="https://avatars2.githubusercontent.com/u/1364564?v=4" width="100px;" alt=""/><br /><sub><b>Ariel Schiavoni</b></sub></a><br /><a href="https://github.com/Astrocoders/reason-apollo-hooks/commits?author=arielschiavoni" title="Documentation">📖</a></td>
    <td align="center"><a href="https://playqup.com"><img src="https://avatars0.githubusercontent.com/u/3103241?v=4" width="100px;" alt=""/><br /><sub><b>Matt</b></sub></a><br /><a href="https://github.com/Astrocoders/reason-apollo-hooks/commits?author=hew" title="Code">💻</a></td>
    <td align="center"><a href="https://twitter.com/_cichocinski"><img src="https://avatars2.githubusercontent.com/u/9558691?v=4" width="100px;" alt=""/><br /><sub><b>Tomasz Cichocinski</b></sub></a><br /><a href="https://github.com/Astrocoders/reason-apollo-hooks/issues?q=author%3Abaransu" title="Bug reports">🐛</a> <a href="https://github.com/Astrocoders/reason-apollo-hooks/commits?author=baransu" title="Code">💻</a></td>
    <td align="center"><a href="https://tmattio.github.io/"><img src="https://avatars0.githubusercontent.com/u/6162008?v=4" width="100px;" alt=""/><br /><sub><b>Thibaut Mattio</b></sub></a><br /><a href="https://github.com/Astrocoders/reason-apollo-hooks/commits?author=tmattio" title="Code">💻</a></td>
    <td align="center"><a href="https://github.com/Emilios1995"><img src="https://avatars1.githubusercontent.com/u/12430923?v=4" width="100px;" alt=""/><br /><sub><b>Emilio Srougo</b></sub></a><br /><a href="https://github.com/Astrocoders/reason-apollo-hooks/issues?q=author%3AEmilios1995" title="Bug reports">🐛</a></td>
    <td align="center"><a href="http://mkndrsn.com"><img src="https://avatars0.githubusercontent.com/u/1226972?v=4" width="100px;" alt=""/><br /><sub><b>Mike Anderson</b></sub></a><br /><a href="https://github.com/Astrocoders/reason-apollo-hooks/commits?author=athaeryn" title="Code">💻</a></td>
  </tr>
  <tr>
    <td align="center"><a href="https://github.com/yurijean"><img src="https://avatars0.githubusercontent.com/u/6414876?v=4" width="100px;" alt=""/><br /><sub><b>Yuri Jean Fabris</b></sub></a><br /><a href="https://github.com/Astrocoders/reason-apollo-hooks/commits?author=yurijean" title="Code">💻</a></td>
    <td align="center"><a href="https://twitter.com/rita_krutikova"><img src="https://avatars2.githubusercontent.com/u/5932274?v=4" width="100px;" alt=""/><br /><sub><b>Margarita Krutikova</b></sub></a><br /><a href="https://github.com/Astrocoders/reason-apollo-hooks/commits?author=MargaretKrutikova" title="Code">💻</a> <a href="https://github.com/Astrocoders/reason-apollo-hooks/pulls?q=is%3Apr+reviewed-by%3AMargaretKrutikova" title="Reviewed Pull Requests">👀</a> <a href="#ideas-MargaretKrutikova" title="Ideas, Planning, & Feedback">🤔</a></td>
    <td align="center"><a href="https://github.com/Yakimych"><img src="https://avatars1.githubusercontent.com/u/5010901?v=4" width="100px;" alt=""/><br /><sub><b>Kyrylo Yakymenko</b></sub></a><br /><a href="https://github.com/Astrocoders/reason-apollo-hooks/issues?q=author%3AYakimych" title="Bug reports">🐛</a> <a href="https://github.com/Astrocoders/reason-apollo-hooks/commits?author=Yakimych" title="Code">💻</a></td>
    <td align="center"><a href="https://github.com/lukashambsch"><img src="https://avatars3.githubusercontent.com/u/7560008?v=4" width="100px;" alt=""/><br /><sub><b>Lukas Hambsch</b></sub></a><br /><a href="https://github.com/Astrocoders/reason-apollo-hooks/issues?q=author%3Alukashambsch" title="Bug reports">🐛</a></td>
    <td align="center"><a href="http://www.familyfive.app"><img src="https://avatars1.githubusercontent.com/u/579279?v=4" width="100px;" alt=""/><br /><sub><b>Jaap Frolich</b></sub></a><br /><a href="https://github.com/Astrocoders/reason-apollo-hooks/commits?author=jfrolich" title="Code">💻</a> <a href="https://github.com/Astrocoders/reason-apollo-hooks/pulls?q=is%3Apr+reviewed-by%3Ajfrolich" title="Reviewed Pull Requests">👀</a> <a href="#ideas-jfrolich" title="Ideas, Planning, & Feedback">🤔</a></td>
    <td align="center"><a href="https://willcodefor.beer/"><img src="https://avatars1.githubusercontent.com/u/1478102?v=4" width="100px;" alt=""/><br /><sub><b>Rickard Laurin</b></sub></a><br /><a href="https://github.com/Astrocoders/reason-apollo-hooks/issues?q=author%3Abeliever" title="Bug reports">🐛</a></td>
    <td align="center"><a href="http://medson.me"><img src="https://avatars0.githubusercontent.com/u/17956325?v=4" width="100px;" alt=""/><br /><sub><b>Medson Oliveira</b></sub></a><br /><a href="https://github.com/Astrocoders/reason-apollo-hooks/commits?author=medson10" title="Code">💻</a> <a href="https://github.com/Astrocoders/reason-apollo-hooks/pulls?q=is%3Apr+reviewed-by%3Amedson10" title="Reviewed Pull Requests">👀</a> <a href="#ideas-medson10" title="Ideas, Planning, & Feedback">🤔</a></td>
  </tr>
  <tr>
    <td align="center"><a href="https://github.com/soulplant"><img src="https://avatars3.githubusercontent.com/u/16846?v=4" width="100px;" alt=""/><br /><sub><b>soulplant</b></sub></a><br /><a href="https://github.com/Astrocoders/reason-apollo-hooks/commits?author=soulplant" title="Code">💻</a></td>
    <td align="center"><a href="https://github.com/mbirkegaard"><img src="https://avatars0.githubusercontent.com/u/18616185?v=4" width="100px;" alt=""/><br /><sub><b>mbirkegaard</b></sub></a><br /><a href="https://github.com/Astrocoders/reason-apollo-hooks/commits?author=mbirkegaard" title="Code">💻</a></td>
    <td align="center"><a href="https://strdr4605.github.io"><img src="https://avatars3.githubusercontent.com/u/16056918?v=4" width="100px;" alt=""/><br /><sub><b>Dragoș Străinu</b></sub></a><br /><a href="https://github.com/Astrocoders/reason-apollo-hooks/commits?author=strdr4605" title="Documentation">📖</a></td>
    <td align="center"><a href="https://github.com/bdunn313"><img src="https://avatars3.githubusercontent.com/u/867683?v=4" width="100px;" alt=""/><br /><sub><b>Brad Dunn</b></sub></a><br /><a href="https://github.com/Astrocoders/reason-apollo-hooks/commits?author=bdunn313" title="Documentation">📖</a></td>
  </tr>
</table>

<!-- markdownlint-enable -->
<!-- prettier-ignore-end -->

<!-- ALL-CONTRIBUTORS-LIST:END -->

This project follows the [all-contributors](https://github.com/all-contributors/all-contributors) specification. Contributions of any kind welcome!
