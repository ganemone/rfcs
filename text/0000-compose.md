* 2019-03-18
* RFC PR: (leave this empty)
* Fusion Issue: (leave this empty)

# Summary

This adds support for composing plugins via `app.compose`, and updates the `createToken` API to
add an optional compose function as the second argument.

# Basic example

```js
const ComposeExampleToken = createToken('ComposeExampleToken', (items) => {
  return items.reduce((sum, item) => {
    return sum + item;
  }, 0);
});

app.compose(ComposeExampleToken, 2);
app.compose(ComposeExampleToken, 3);

app.register(createPlugin({
  deps: {
    composeExample: ComposeExampleToken
  },
  provides: ({composeExample}) {
    console.log(composeExample); // 5
  }
}));
```

# Motivation

There are often situations where you need to manage composing a large amount of objects onto a single token. This can be done today with a few approaches today,
but they all require significant boilerplate and often unreasonable overheard. For example, you may want to split your graphql schemas and resolvers up into many
separate modular schemas, and combine them onto a single token using schema stiching. However as the number of separate schemas expands upwards of 10, 20, it 
becomes restrictive to manage separate tokens for each separate schema.

## Use Cases


A real life use case of `app.compose` would be managing graphql schemas.

```js
// Old way
app.register(TokenA, SchemaA);
app.register(TokenB, SchemaB);
app.register(TokenC, SchemaC);
app.register(TokenD, SchemaD);
app.register(TokenE, SchemaE);

app.register(SchemaToken, createPlugin({
  deps: {
    a: TokenA,
    b: TokenB,
    c: TokenC,
    d: TokenD,
    e: TokenE,
  },
  provides({a, b, c, d, e}) {
    return mergeSchemas([a,b,c,d,e])
  }
}))
```

This becomes exceedingly difficult as the number of schemas increases. With the app.compose method, this could be replaced with:

```js
// new way
const SchemaToken = createToken('SchemaToken', mergeSchemas);
app.compose(SchemaToken, SchemaA);
app.compose(SchemaToken, SchemaB);
app.compose(SchemaToken, SchemaC);
app.compose(SchemaToken, SchemaD);
app.compose(SchemaToken, SchemaE);
```

Another real life use case could be managing redux store enhancers.

```js
app.compose(ReduxEnhancerToken, enhancerA)
app.compose(ReduxEnhancerToken, enhancerB)
app.compose(ReduxEnhancerToken, enhancerC)
```

# Detailed design

This RFC proposes two nonbreaking API changes which would support managing plugin composition.

1. `createToken` will take a second optional paramter: composeFn which will be used in the case where plugins are registered via `app.compose`.
2. `app.compose` will be added to the app class which will have the same type signature as `app.register`.

app.compose will throw if the provided token does not have a compose function. During the dependency resolution process, all plugins registered via app.compose
will be resolved by calling the corresponding compose function. 

app.register overrides any previous values registered on the token, same as its current behavior. 

app.compose overrides a previously registered value via app.register.

Enhancers apply to composed registrations after composition.

- `app.register(TokenA, PluginA)` => TokenA is implemented by PluginA. It may be enhanced in the future, or overwritten via the DI system.
- `app.compose(TokenA, PluginA)` => PluginA will be composed with other Plugins registered onto TokenA. They may be enhanced in the future, or overwritten via the DI system
- `app.enhance(TokenA, PluginA)` => PluginA will enhance the existing value on TokenA. This can be commonly used for things like adding caching wrappers. The enhancer is not an implementation of the token.

# Drawbacks

Increased API surface area is the biggest drawback IMO. The addition of this API does create situations where there are multiple possible
ways of setting up various DI graphs. 

# Alternatives

One alternative is continuing with the current model of managing plugin composition by putting each item onto unique tokens.
