# Layered Configs
A tool for managing overrideable Javascript values at runtime, handy as a config store.

**THIS LIBRARY IS IN ACTIVE DEVELOPMENT, AND IS NEITHER RELEASED NOR FUNCTIONAL**

Everything below this point is hypothetical future work.

## What is it?

A tool for managing potentially-overrideable values in a structured way. Intended as a very
flexible tool that still has enough constraints to force consistency app-wide, and to catch
potential errors more easily.

* Define base default values, along with the manners ("layers") that overrides can be applied by.
* Enable overriding layers to override sets of values (or disable them to revert to whatever was
    there before), at runtime.
* Apply arbitrary values to layers dynamically.
* Config values can be derived from other config values, as aliases or expressions or templates,
    to encapsulate complex internal behavior where appropriate.
* Consumers only see the resulting values, not the causes or logic that went into making them.

## Example

```javascript
const backendConfig = createLayeredConfig({
  label: 'backendConfig',
  layers: [
    {
      label: 'localDev',
      readOnly: true,
      initialValues: {
        hostname: 'localhost',
        apiVersion: 2,
        
        baseUrl: '$$("https://" + hostname + "/api/v" + apiVersion + "/")',
        // Or you could do it this way
        baseUrl: '$$`https://${hostname}/api/v${apiVersion}/`',
        // Or even this
        baseUrl: 'tmpl:https://<%= hostname %>/api/v<%= apiVersion %>',
      },
    },
    {
      label: 'devStaging',
      isEnabled: /(dev|staging|preview).example.com/.test(window.location.hostname),
      initialValues: {
        hostname: window.location.hostname,
        // The other values -- like apiVersion and baseUrl -- are inherited, and when their
        // expressions or templates are evaluated they'll receive this hostname value.
      },
    },
    {
      label: 'prod',
      isEnabled: window.location.hostname === 'www.example.com',
      readOnly: true,
      initialValues: {
        hostname: 'www.example.com',
        apiVersion: 1,
      },
    },
    {
      label: 'debugOverride',
      // This gives us a place to set any arbitrary values we want at runtime, to allow devs and
      // QA to set whatever values they desire, without affecting the presets above.
      // For example, you could add a secret toggle that sets `{ apiVersion: 2 }` here, and then
      // test that new apiVersion on prod without exposing all users to v2.
    },
  ],
});

// Then, you can do things like:
backendConfig.enableLayer('prod'); // although this is unnecessary, since it self-enables above

backendConfig.assignLayerValues('debugOverride', { apiVersion: 2 });

backendConfig.get('baseUrl'); // Or you can use a path or array if it's a deep path
// => 'https://www.example.com/api/v2/'
```

## How does it work?

There are two general ways to model config or environment values:

* You could have *one* active config, and allow it to extend or inherit from a specific parent:
    "production" inherits from "base". Essentially, start from one child and work towards a root.
    *That's not what this is*, although many good config systems work this way.
* You could have a base schema with some defaults, and apply a series of overrides to it:
    "base" gets masked over by "production", and possibly by additional things as well. That's
    the model this tool follows.

The general idea is to model the "causes" or "factors" that drive config values as different
"layers". Internally, the various causes/factors manipulate the resulting state in some way:
maybe they merely override a value, maybe they have a heavier effect.

Regardless of what they do internally, though, once the causes/factors are finished, you end up
with *one* config: it's effectively just a set of values, and it represents the things consumers
should care about, without regard for any of the internal causes or logic which might affect
that value.

### General algorithm

1. Collapse the raw layers down to a single object.
    1. Start with the lowest layer: it provides default values, and sets the schema for what
        the other layers are permitted to override.
    2. For each other layer (if it's enabled), merge/apply its values on top of the accumulated
        results so far. By default, `null` and `undefined` allow earlier values to pass through.
    3. Once all of the layers have been applied, we're left with a single object of all the raw
        values, without really caring whether they're defaults or overrides.
        
2. Apply aliases, expressions, and templates.
    * When a specific config values is requested, if it's a string:
        ```
        if (it starts with the "expression" prefix) { // '$$' by default
          Parse the expression and figure out what it means
          Resolve it to a single value (recursing to get other values, if necessary)
        } else if (it starts with the "template" prefix) { // 'tmpl:' by default
          Compile it as a template
          Resolve any referenced values (recursing to get them, if necessary)
          Evaluate the template with the final data
        }
        Now that we have a value, apply any further alias|exrpession|template work (again,
        recursively if necessary), and then we have our final value
        ```
    * If a non-leaf group was requested, we'll do the above to get the object as a whole, and then
        also resolve each child within the group.

### Value Depth

For consistency and developer sanity, and because it often helps detect and prevent bugs, config
values are *always* stored at the same depth -- 1 key deep, by default -- for all values in the
config.

Although the default `valueDepth` is 1, for a nontrivial app you may want to increase that to 2,
to force each value to live at `key1.key2`: this lets you group things by their relevance or
meaning, and makes it considerably easier to scale over time.

```javascript
const backendConfig = createLayeredConfig({
  label: 'backendConfig',
  valueDepth: 2,
  layers: [
    {
      label: 'localDev',
      readOnly: true,
      initialValues: {
        api: {
          hostname: 'localhost:4000',
          apiVersion: 2,
          // Use a similar baseUrl as above, but use "api.hostname" instead of "hostname"
        },
        assets: {
          hostname: 'localhost:3000',
          // this can get a completely different baseUrl pattern
          baseUrl: '$$`https://${hostname}/assets/`',
          
        },
        // And with a pattern like this, we can now support multiple hosts with their own config,
        // or semi-shared configs if that's what you need. This works well for microservices.
```

## Options

Note again that this is hypothetical, and is being reimplemented from scratch without using prior
company resources.

### createConfig(options)

This is the main entry point for defining a Layered Config store.

Name | Type | Default | Description
---- | ---- | ------- | -----------
label | String | (required) | A string to use in debug and error messages to describe this set of config values, and (potentially) to look it up globally.
schemaVersion | Number | `1` | To manage storage schemas, set and increment this value whenever you introduce a breaking change in your own code.
valueDepth | Number | `1` | To scale up to non-flat objects over time, set this to the number of keys you intend each value to be accessed/indicated/addressed by. ALL values in the config MUST live at this depth.
postProcessExpressionPrefix | String | `"$$"` | If a config value begins with this string, it will be interpreted as an expression, with any values resolved from the config.
postProcessTemplatePrefix | String | `"tmpl:"` | If a config value begins with this string, it will be interpreted as a template, with any values resolved from the config.
postProcessTemplateCompiler | Function | `(templateString) => lodash.template(templateString)` | Whenever the source template changes (as resolved after layers are applied), this can compile it into a reusable function. Useful if template application is expensive or requires prerequisite work.
postProcessTemplateEvaluator | Function | `(compiledTemplate, data) => value` | Whenever either the a string template or its dependencies change, this processes the template with its requisite data.
treatNullAsUndefined | Boolean | `true` | Whether to consider `null` values in an individual layer to be the same as `undefined` (i.e., whether the layer should override earlier values with `null` or whether to not apply an override). Changing this allows `null` to be an explicit overriding value, instead of a way to indicate you don't wish to override, although in general this is not recommended.
callbacksEnabled | Boolean | `__DEV__` if defined, else `true` | Whether to allow any of the `onWhatever` callbacks below to fire.
onDevSuggestion | Function | `console.warn` | Will be called if your `createConfig` or `layers` config does not match recommendations and best practices.
onInvalidValueDepth | Function | `console.error` | Will be called if a layer attempts to set a config value at any depth other than `valueDepth`.
onOverrideIsMissingBase | Function | `console.error` | Will be called if a layer attempts to set a config value which was not given a default value in the lowest-level base default.
onOverrideHasDifferentType | Function | `console.error` | Will be called if a layer attempts to set a config value whose `typeof` is different from the value it's overriding.
onSetValueInDisabledLayer | Function | `console.warn` | Will be called if you assign values to a layer which is disabled. You probably want to call `yourConfig.enableLayer('layerLabel')` first.
onStorageNotRestored | Function | `console.warn` | Will be called if `restoreFromStorage` does not return either `true` or an explicit internal state to use.
onAnyChange | Function | `null` | Will be called anytime anything changes in any layer, in addition to any layer-specific callbacks.
onGetValue | Function | `null` | Will be called anytime any consumer requests a value from the resulting config. This can be quite noisy.
layers | Array | (required) | An array of descriptors for the allowed "layers". See the next section ("Each config layer") for schema.

### Each config layer (in `options.layers`)

Name | Type | Default | Description
---- | ---- | ------- | -----------
layerName | String | (required) | A string to use in debug and error messages to describe this config layer.
initialValues | Object | `{}` | The initial set of values applied by this layer.
readOnly | Boolean | `false` | Whether to allow consumers to override the `initialValues` after initialization (via `assignLayerValues`, `replaceLayerValues, or `clearLayerValues`)
isEnabled | Boolean | `!readOnly` | Whether this layer's values are applied by default, or not. Unless specified, readOnly layers are *disabled* by default (since they presumably include a set of values to applied in some specific case) while read-write layers are *enabled* by default (since they will presumably be populated dynamically only once needed).
callbacksEnabled | Boolean | `__DEV__` if defined, else `true` | Whether to allow any of the `onWhatever` callbacks below to fire. Note that this does not affect the config-level callbacks.
onAnyChange | Function | `null` | Will be called anytime anything changes, either values or status, in addition to the more speicific callbacks.
onValueChange | Function | `console.log` | Will be called when this layer has values assigned/replaced/cleared.
onStatusChange | Function | `console.log` | Will be called when this layer is enabled or disabled, or if new callbacks are set for it.

## API

### Constructor

**`createConfig(options)`** is documented above.

**`const myOwnConfigCreator = createConfig.withOptions(myOwnOptions)`** lets you create your own
    `createConfig` with different options applied over the defaults. This is similar to using a
    partial function to set argument values, except it applies to keys within `options`. There is
    no true "global defaults" system: applying customizations via your own wrapper is recommended.

### On the store instance

These live on the config store instance itself

(docs/readme is unrevised below this point)

`enableCallbacks()`/`disableCallbacks()`/`setCallbacksEnabled(isEnabled)` enables or disables ALL
    callbacks in ALL layers of the config.

`enableLayerCallbacks(label)`/`disableLayerCallbacks(label)`/`setLayerCallbacksEnabled(label, isEnabled)`
    enables or disables the callbacks for the specific layer of the config.

`config.onWhatever(newFn)`, `config.onWhatever(layer, newFn)`: Overrides any of the `onWhatever`
    callbacks defined above. Returns a function that will revert it to its *exact* prior value.

`enableLayer(label)`/`disableLayer(label)`/`setLayerEnabled(layer, isEnabled)` enables or disables
    the values in the specified label.

`assignLayerValues(layer, values)` merges new override values for the specified layer.

`replaceLayerValues(layer, values)` replaces all override values for the specified layer.

`clearLayerValues(layer)` remove all override values for the specified layer.

`get('some.path')`/`get('some', 'path')`/`get(['some', 'path'])` returns a value from the config
    store, after applying layers, overrides, and any aliases/expressions/templates during
    post-processing.

`serializeForStorage(additionalMetaData = null)` returns a string representation of the layered
    config's internal state.

`restoreFromStorage(serializedData, callback(layerValuesBeforeApplied, additionalMetaData))`
    lets the config's internal state be initialized from storage when appropriate.
    `layerValuesBeforeApplied` can be mutated or replaced by returning a new value, although this
    is dangerous and generally not recommended. If the storage is for an old schema, or if this
    callbacks returns in invalid or incompatible value, `onStorageNotRestored` will be fired.

