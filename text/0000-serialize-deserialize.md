* Start Date: 2018-08-01
* RFC PR: 
* Fusion Issue: 

# Summary

This RFC describes a process for standardizing on an API for serializing and deserializing state between server and browser plugins.

# Basic example

```js
import {SharedStateToken, createPlugin} from 'fusion-core';

export default createPlugin({
  deps: {
    SharedState: SharedStateToken,
  },
  middleware: ({SharedState}) => {
    const {serialize, deserialize} = SharedState('some-id');
    return (ctx, next) => {
      if (__BROWSER__) {
        // load and parse serialized data from dom node
        const data = deserialize();
      } else {
        // serialize json data into body
        serialize(ctx, {
          some: 'json',
        });
      }
      return next();
    };
  },
});

// in tests
import {SharedStateToken, MockSharedState} from 'fusion-core';
app.register(SharedStateToken, MockSharedState({
  'some-id': {
    some: 'json'
  },
  'other-data': {}
}));
```

# Motivation

It is a very common pattern in fusion to have some shared state between the server and
the browser encapsulated into a plugin. The current model for this has users referencing
`ctx.template` directly on the server, and querying the DOM directly from the browser.

This can make testing difficult, especially when running browser tests since there is not
an easy way to mock the deserialization other than each plugin exposing their own deserialization
token. 

Providing a standard API for sharing json data between server and browser plugins
will make it easier to test these types of plugins in isolation without needing to resort 
to integration tests. Additionally, it can help solve problems where users may want custom 
serialization / deserialization mechanisms, such as using tools like immutable.js.

# Detailed design

This proposal adds the following APIs to `fusion-core` 

### `SharedStateToken`

```js
import {SharedStateToken} from 'fusion-core';
```

The `SharedStateToken` will have a default implementation registered onto each fusion app
which provides a `{serialize}` and `{deserialize}` function which serialize json data on the
server into a `<script type='application/json'>` dom element which is appended to `ctx.template.body`, and
deserialize the data from the dom node in the browser. Example Usage:

```js
import {SharedStateToken, createPlugin} from 'fusion-core';

export default createPlugin({
  deps: {
    SharedState: SharedStateToken,
  },
  middleware: ({SharedState}) => {
    const {serialize, deserialize} = SharedState('some-id');
    return (ctx, next) => {
      if (__BROWSER__) {
        // load and parse serialized data from dom node
        const data = deserialize();
      } else {
        // serialize json data into body
        serialize(ctx, {
          some: 'json',
        });
      }
      return next();
    };
  },
});
```

### `MockSharedState`

```js
import {MockSharedState} from 'fusion-core';
```

This API will provide an easy way to provide mock data for all deserialize
calls in an application. 

```js
import {MockSharedState, SharedStateToken} from 'fusion-core';
// Deserialize calls will now return the mock values for these ids
app.register(SharedStateToken, MockSharedState({
  'some-id': {
    some: 'json'
  },
  'other-data': {}
}));
```

### Custom Serialization

This structure can also be useful when trying to implement custom serialization / deserialization for certain plugins. 
For example, lets say you want to use the `fusion-plugin-react-redux` plugin but want to have custom serialization / deserialization
to support immutable-js in your store. You can create your own custom serializer/deserializer, and even set up your app to only
use this custom SharedState implementation for the react redux plugin using token aliasing.

```js
import {createToken, html, SharedStateToken} from 'fusion-core';
import ReduxPlugin, {ReduxToken} from 'fusion-plugin-react-redux';
import transit from 'transit-immutable-js';

const ImmutableSharedState = (id) => {
  return {
    serialize: (ctx, data) => {
      ctx.template.body.push(
        html`<div id='${id}'>${transit.toJSON(data)}</div>`
      );
    },
    deserialize: () => {
      return transit.fromJSON(document.getElementById(id).textContent);
    }
  }
}
const ImmutableSharedStateToken = createToken('ImmutableSharedState');

app.register(ImmutableSharedStateToken, ImmutableSharedState);
// Redux will now use the immutable serializer, while the rest of your plugins will use the standard serializer
app.register(ReduxToken, ReduxPlugin)
  .alias(SharedStateToken, ImmutableSharedStateToken);
```

# Drawbacks

* Does not handle serialization of non json data types. For example, injecting styles into head
* Does not provide an API for putting the data into head vs body

Rather than try and build an all encompassing API, I purposely scoped this API to sharing json data because
I think it is the most common pattern which requires both serialization and deserialization.  

# Alternatives

There are many variations of this API we should consider. One potential option would be to have a `ctx.serialize` and `ctx.deserialize` function,
and provide a way to mock those functions in testing via something in `fusion-test-utils`. This could solve the issue with testing, but would
require introducing new APIs to `fusion-test-utils`. While the above proposal does introduce new Tokens, it uses the DI system to solve the 
problem so no new core concepts need to be introduced. Also, using the DI system allows us to simultaneously solve the issue of custom serialization / deserialization functions.

# Adoption strategy

This is a feature add, with no backwards incompatible changes necessary. Plugins can continue manually serializing / deserializing data, however
it should be encouraged to follow this pattern. We should update all the plugins we own to use this new pattern. 

# How we teach this

We should provide documentation on the new APIs introduced here, and can link to this RFC for a more in depth explanation on the benefits of having
serialization / deserialization go through the DI system.
