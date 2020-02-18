---
title: API
type: guide
order: 2
dropdown: extensions
repo: batch-loader
---

<!--- Usage --------------------------------------------------------------------------------------->
<h2 id="Usage">Usage</h2>

``` js
npm install --save @feathers-plus/batch-loader

// JS
const BatchLoader = require('@feathers-plus/batch-loader');
const { getResultsByKey, getUniqueKeys } = BatchLoader;

const usersLoader = new BatchLoader(async (keys, context) => {
    const usersRecords = await users.find({ query: { id: { $in: getUniqueKeys(keys) } } });
    return getResultsByKey(keys, usersRecords, user => user.id, '')
  },
  { context: {} }
);

const user = await usersLoader.load(key);
```

> May be used on the client.

<!--- class BatchLoader --------------------------------------------------------------------------->
<h2 id="class-batchloader">class BatchLoader( batchLoadFunc [, options] )</h2>

Create a new batch-loader given a batch loading function and options.

- **Arguments:**
  - `{Function} batchLoadFunc`
  - `{Object} [ options ]`
    - `{Boolean} batch`
    - `{Boolean} cache`
    - `{Function} cacheKeyFn`
    - `{Object} cacheMap`
    - `{Object} context`
    - `{Number} maxBatchSize`

Argument | Type | Default | Description
---|:---:|---|---
`batchLoadFunc` | `Function` | | See [Batch Function](guide.html#batch-function).
`options` | `Object` | | Options.

`options` | Argument | Type | Default | Description
---|---|---|---|---
 | `batch` | Boolean | `true` | Set to false to disable batching, invoking `batchLoadFunc` with a single load key.
 | `cache` | Boolean | `true` | Set to false to disable memoization caching, creating a new Promise and new key in the `batchLoadFunc` for every load of the same key.
 | `cacheKeyFn` | Function | `key => key` | Produces cache key for a given load key. Useful when keys are objects and two objects should be considered equivalent.
 | `cacheMap` | Object | `new Map()` | Instance of Map (or an object with a similar API) to be used as cache. See below.
 | `context` | Object | `null` | A context object to pass into `batchLoadFunc` as its second argument.
 | `maxBatchSize` | Number | `Infinity` | Limits the number of keys when calling `batchLoadFunc`.

{% apiReturns class-batchloader "BatchLoader instance." batchLoader Object %}

- **Example**

  ``` js
  const BatchLoader = require('@feathers-plus/batch-loader');
  const { getResultsByKey, getUniqueKeys } = BatchLoader;

  const usersLoader = new BatchLoader(async (keys, context) => {
      const data = await users.find({ query: { id: { $in: getUniqueKeys(keys) } }, paginate: false });
      return getResultsByKey(keys, data, user => user.id, '')
    },
    { context: {}, batch: true, cache: true }
  );
  ```

- **Pagination**

  The number of results returned by a query using `$in` is controlled by the pagination `max` set for that Feathers service. You need to specify a `paginate: false` option to ensure that records for all the keys are returned.
   
  The maximum number of keys the `batchLoadFunc` is called with can be controlled by the BatchLoader itself with the `maxBatchSize` option. 

- **option.cacheMap**

  The default cache will grow without limit, which is reasonable for short lived batch-loaders which are rebuilt on every request. The number of records cached can be limited with a  *least-recently-used* cache:
  
  ``` js
  const BatchLoader = require('@feathers-plus/batch-loader');
  const cache = require('@feathers-plus/cache');
  
  const usersLoader = new BatchLoader(
    keys => { ... },
    { cacheMap: cache({ max: 100 })
  );
  ```
    
  > You can consider wrapping npm's `lru` on the browser.
  
- **See also:** [Guide](./guide.html)

{% apiFootnote classBatchLoader BatchLoader %}

<!--- loaderFactory ------------------------------------------------------------------------------->
<h2 id="loader-factory">static BatchLoader.loaderFactory( service, id, multi, options  )</h2>

Helper method to create basic loaders.

- **Arguments:**
  - `{Object} service`
  - `{String} id`
  - `{Boolean} multi`
  - `{Object} [ options ]`
    - `{Function} getKey`
    - `{Array<String>} paramNames`
    - `{Object} injects`
  
Argument | Type | Default | Description
---|---|---|---
`service` | `Object` | | The service of the records to be fetched.
`id` | `String` | | The name of the unique property on the record to be fetched. For example, `'id'` or `'_id'`.
`multi` | `Boolean` | `false` | Should the loader be allowed to return multiple records. This is shorthand for the `'[!]'` or `'!'` syntax
`options` | `Object` | `{}` | Options

`options` | Argument | Type | Default | Description
---|---|---|---|---
 | `getKey` | Function | `rec => rec[id]` | Function to resolve the corresponding `id` parameter on each record. Passed to `getResultsByKey` under the hood.
 | `paramNames` | [String] | | String param names to be kept from context and passed to service call. See [source](https://github.com/feathers-plus/batch-loader/blob/e22aaaac8766f6b35bbfe19718f928106521ca50/lib/index.js#L286)
 | `injects` | Object | | Object of params to be added to service call. See [source](https://github.com/feathers-plus/batch-loader/blob/e22aaaac8766f6b35bbfe19718f928106521ca50/lib/index.js#L286)

{% apiReturns class-batchloader "BatchLoader instance." batchLoader Object %}

- **Example:**

  ``` js
  const usersLoader = BatchLoader.loaderFactory(app.service('users'), 'id');
  ```
  
- **Details**

  The loaderFactory method creates a simple batchLoader that is suitable for most cases. This simple one-liner creates the same batchLoader instance as the example for the `class BatchLoader` above.
    
  
{% apiFootnote getUniqueKeys BatchLoader %}
<!--- getUniqueKeys ------------------------------------------------------------------------------->
<h2 id="get-unique-keys">static BatchLoader.getUniqueKeys( keys )</h2>

Returns the unique elements in an array.

- **Arguments:**
  - `{Array<String | Number>} keys`
  
Argument | Type | Default | Description
---|---|---|---
`keys` | `Array<` `String /` `Number >` | | The keys. May contain duplicates.

{% apiReturns get-unique-keys "The keys with no duplicates." keys "Array< String / Number >" %}

- **Example:**

  ``` js
  const usersLoader = new BatchLoader(async keys =>
    const data = users.find({ query: { id: { $in: getUniqueKeys(keys) } } })
    ...
  );
  ```
  
- **Details**

  The array of keys may contain duplicates when the batch-loader's memoization cache is disabled.
    
  <p class="tip">Function does not handle keys of type Object nor Array.</p>
  
{% apiFootnote getUniqueKeys BatchLoader %}

<!--- getResultsByKey ----------------------------------------------------------------------------->
<h2 id="get-results-by-key">static BatchLoader.getResultsByKey( keys, records, getRecordKeyFunc, type [, options] )</h2>

Reorganizes the records from the service call into the result expected from the batch function.

- **Arguments:**
  - `{Array<String | Number>} keys`
  - `{Array<Object>} records`
  - `{Function} getRecordKeyFunc`
  - `{String} type`
  - `{Object} [ options ]`
    - `{null | []} defaultElem`
    - `{Function} onError`

Argument | Type | Default | Description
---|:---:|---|---
`keys` | `Array<` `String /` `Number>` | | An array of `key` elements, which the value the batch loader function will use to find the records requested.
`records` | `Array< ` `Object >` | | An array of records which, in total, resolve all the `keys`.
`getRecordKeyFunc` | `Function` | | See below.
`type` | `String` | | The type of value the batch loader must return for each key.
`options` | `Object` | | Options.

`type` | Value | Description
---|:---:|---
 | `''` | An optional single record.
 | `'!'` | A required single record.
 | `'[]'` | A required array including 0, 1 or more records.

`options` | Argument | Type | Default | Description
---|---|:---:|---|---
 | `defaultElem` | `{null / []}` | `null` | The value to return for a `key` having no record(s). 
 | `onError` | `Function` | `(i, msg) => {}` | Handler for detected errors, e.g. `(i, msg) =>` `{ throw new Error(msg,` `'on element', i); }`
 
{% apiReturns get-results-by-key "The result for each key. <code>results[i]</code> is the result for <code>keys[i]</code>." results "Array< Object >"%}
  
- **Example**

  ``` js
    const usersLoader = new BatchLoader(async keys => {
      const data = users.find({ query: { id: { $in: getUniqueKeys(keys) } } })
      return getResultsByKey(keys, data, user => user.id, '', { defaultElem: [] })
    });
  ```

- **Details**  

  <p class="tip">Function does not handle keys of type Object nor Array.</p>
  
- **getRecordKeyFunc**

  A function which, given a record, returns the key it satisfies, e.g.
  ``` js
  user => user.id
  ```

- **See also:** [Batch-Function](./guide.html#Batch-Function)

{% apiFootnote getResultsByKey BatchLoader %}

<!--- load ---------------------------------------------------------------------------------------->
<h2 id="load">batchLoader.load( key )</h2>

Loads a key, returning a Promise for the value represented by that key.

- **Arguments:**
  - `{String | Number | Object | Array} key`

Argument | Type | Default | Description
---|:---:|---|---
`key` | `String` `Number` `Object` `Array` | | The key the batch-loader uses to find the result(s).

{% apiReturns load "Resolves to the result(s)." promise "Promise< Object >"%}

- **Example:**

  ``` js
  const batchLoader = new BatchLoader( ... );
  const user = await batchLoader.load(key);
  ```

{% apiFootnote load BatchLoader %}

<!--- loadMany ------------------------------------------------------------------------------------>
<h2 id="loadmany">batchLoader.loadMany( keys )</h2>

Loads multiple keys, promising a arrays of values. 

- **Arguments**
  - `{Array<String | Number | Object | Array>} keys`

Argument | Type | Default | Description
---|:---:|---|---
`keys` | `Array<` `String /` ` Number /` ` Object /` ` Array>` | | The keys the batch-loader will return result(s) for.

{% apiReturns load "Resolves to an array of result(s). <code>promise[i]</code> is the result for <code>keys[i]</code>." promise "Promise[ Array< Object > ]"%}

- **Example**

  ``` js
  const usersLoader = new BatchLoader( ... );
  const users = await usersLoader.loadMany([ key1, key2 ]);
  ```

- **Details**
 
  This is a convenience method. `usersLoader.loadMany([ key1, key2 ])` is equivalent to the more verbose:
  ``` js
  Promise.all([
    usersLoader.load(key1),
    usersLoader.load(key2)
  ]);
  ```

{% apiFootnote loadMany BatchLoader %}

<!--- clear --------------------------------------------------------------------------------------->
<h2 id="clear">batchLoader.clear( key )</h2>

Clears the value at key from the cache, if it exists.

- **Arguments:**
  - `{String | Number | Object | Array} key`

Argument | Type | Default | Description
---|:---:|---|---
`key` | `String` `Number` `Object` `Array` | | The key to remove from the cache.

- **Details**

The key is matches using strict equality. This is particularly important for `Object` and `Array` keys.

{% apiFootnote clear BatchLoader %}

<!--- clearAll ------------------------------------------------------------------------------------>
<h2 id="clearall">batchLoader.clearAll()</h2>

Clears the entire cache. 

- **Details**

  To be used when some event results in unknown invalidations across this particular batch-loader.

{% apiFootnote clearAll BatchLoader %}

<!--- prime --------------------------------------------------------------------------------------->
<h2 id="prime">batchLoader.prime( key, value )</h2>

Primes the cache with the provided key and value. 

- **Arguments:**
  - `{String | Number | Object | Array} key`
  - `{Object} record`

Argument | Type | Default | Description
---|:---:|---|---
`key` | `String` `Number` `Object` `Array` | | The key in the cache for the record.
`record` | `Object` | | The value for the `key`.

- **Details**

  **If the key already exists, no change is made.** To forcefully prime the cache, clear the key first with `batchloader.clear(key)`.

{% apiFootnote prime BatchLoader %}

<!--- What's New --------------------------------------------------------------------------->
<h2 id="whats-new">What's New</h2>

The details are at <a href="https://github.com/feathers-plus/batch-loader/blob/master/CHANGELOG.md">Changelog.</a>

#### Feb. 2018

- Added information about pagination within the `batchLoadFunc`.
  
