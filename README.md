# fastify-rate-limit

![CI](https://github.com/fastify/fastify-rate-limit/workflows/CI/badge.svg)
[![NPM version](https://img.shields.io/npm/v/fastify-rate-limit.svg?style=flat)](https://www.npmjs.com/package/fastify-rate-limit)
[![Known Vulnerabilities](https://snyk.io/test/github/fastify/fastify-rate-limit/badge.svg)](https://snyk.io/test/github/fastify/fastify-rate-limit)
[![js-standard-style](https://img.shields.io/badge/code%20style-standard-brightgreen.svg?style=flat)](https://standardjs.com/)

A low overhead rate limiter for your routes. Supports Fastify `2.x - 3.x` semver range.

Please refer to [this branch](https://github.com/fastify/fastify-rate-limit/tree/1.x) and related versions for Fastify 1.x compatibility.

## Install
```
npm i fastify-rate-limit
```

## Usage
Register the plugin and, if required, pass some custom options.<br>
This plugin will add an `onRequest` hook to check if a client (based on their IP address) has made too many requests in the given timeWindow.
```js
const fastify = require('fastify')()

fastify.register(require('fastify-rate-limit'), {
  max: 100,
  timeWindow: '1 minute'
})

fastify.get('/', (request, reply) => {
  reply.send({ hello: 'world' })
})

fastify.listen(3000, err => {
  if (err) throw err
  console.log('Server listening at http://localhost:3000')
})
```

In case a client reaches the maximum number of allowed requests, an error will be sent to the user with the status code set to `429`:
```js
{
  statusCode: 429,
  error: 'Too Many Requests',
  message: 'Rate limit exceeded, retry in 1 minute'
}
```
You can change the response by providing a callback to `errorResponseBuilder` or setting a [custom error handler](https://www.fastify.io/docs/latest/Server/#seterrorhandler):

```js
fastify.setErrorHandler(function (error, request, reply) {
  if (reply.statusCode === 429) {
    error.message = 'You hit the rate limit! Slow down please!'
  }
  reply.send(error)
})
```

The response will have some additional headers:

| Header | Description |
|--------|-------------|
|`x-ratelimit-limit`     | how many requests the client can make
|`x-ratelimit-remaining` | how many requests remain to the client in the timewindow
|`x-ratelimit-reset`     | how many seconds must pass before the rate limit resets
|`retry-after`           | if the max has been reached, the milliseconds the client must wait before they can make new requests


### Preventing guessing of URLS through 404s

An attacker could search for valid URLs if your 404 error handling is not rate limited.
To rate limit your 404 response, you can use a custom handler:

```js
const fastify = Fastify()
await fastify.register(rateLimit, { global: true, max: 2, timeWindow: 1000 })
fastify.setNotFoundHandler({
  preHandler: fastify.rateLimit()
}, function (request, reply) {
  reply.code(404).send({ hello: 'world' })
})
```

Note that you can customize the behaviour of the preHandler in the same way you would for specific routes:

```js
const fastify = Fastify()
await fastify.register(rateLimit, { global: true, max: 2, timeWindow: 1000 })
fastify.setNotFoundHandler({
  preHandler: fastify.rateLimit({
    max: 4,
    timeWindow: 500
  })
}, function (request, reply) {
  reply.code(404).send({ hello: 'world' })
})
```

### Options

You can pass the following options during the plugin registration:
```js
fastify.register(require('fastify-rate-limit'), {
  global : false, // default true
  max: 3, // default 1000
  ban: 2, // default null
  timeWindow: 5000, // default 1000 * 60
  hook: 'preHandler', // default 'onRequest'
  cache: 10000, // default 5000
  allowList: ['127.0.0.1'], // default []
  redis: new Redis({ host: '127.0.0.1' }), // default null
  continueExceeding: true, // default false
  skipOnError: true, // default false
  keyGenerator: function (request) { /* ... */ }, // default (request) => request.raw.ip
  errorResponseBuilder: function (request, context) { /* ... */},
  enableDraftSpec: true, // default false. Uses IEFT draft header standard
  addHeadersOnExceeding: { // default show all the response headers when rate limit is not reached
    'x-ratelimit-limit': true,
    'x-ratelimit-remaining': true,
    'x-ratelimit-reset': true
  },
  addHeaders: { // default show all the response headers when rate limit is reached
    'x-ratelimit-limit': true,
    'x-ratelimit-remaining': true,
    'x-ratelimit-reset': true,
    'retry-after': true
  }
})
```

- `global` : indicates if the plugin should apply the rate limit setting to all routes within the encapsulation scope
- `max`: is the maximum number of requests a single client can perform inside a timeWindow. It can be an async function with the signature `async (request, key) => {}` where `request` is the Fastify request object and `key` is the value generated by the `keyGenerator`. The function **must** return a number.
- `ban`: is the maximum number of 429 responses to return to a single client before returning 403. When the ban limit is exceeded the context field will have `ban=true` in the errorResponseBuilder. This parameter is an in-memory counter and could not work properly in a distributed environment.
- `timeWindow:` the duration of the time window. It can be expressed in milliseconds or as a string (in the [`ms`](https://github.com/zeit/ms) format)
- `cache`: this plugin internally uses a lru cache to handle the clients, you can change the size of the cache with this option
- `allowList`: array of string of ips to exclude from rate limiting. It can be a sync function with the signature `(request, key) => {}` where `request` is the Fastify request object and `key` is the value generated by the `keyGenerator`. If the function return a truthy value, the request will be excluded from the rate limit.
- `redis`: by default this plugins uses an in-memory store, which is fast but if you application works on more than one server it is useless, since the data is stored locally.<br>
You can pass a Redis client here and magically the issue is solved. To achieve the maximum speed, this plugin requires the use of [`ioredis`](https://github.com/luin/ioredis). **Note:** the [default parameters](https://github.com/luin/ioredis/blob/v4.16.0/API.md#new_Redis_new) of a redis connection are not the fastest to provide a rate-limit. We suggest to customize the `connectTimeout` and `maxRetriesPerRequest` as in the [`example`](https://github.com/fastify/fastify-rate-limit/tree/master/example/example.js).
- `continueExceeding`: Renew user limitation when user sends a request to the server when still limited
- `store`: a custom store to track requests and rates which allows you to use your own storage mechanism (using an RDBMS, MongoDB, etc.) as well as further customizing the logic used in calculating the rate limits. A simple example is provided below as well as a more detailed example using Knex.js can be found in the [`example/`](https://github.com/fastify/fastify-rate-limit/tree/master/example) folder
- `skipOnError`: if `true` it will skip errors generated by the storage (e.g. redis not reachable).
- `keyGenerator`: a function to generate a unique identifier for each incoming request. Defaults to `(request) => request.ip`, the IP is resolved by fastify using `request.connection.remoteAddress` or `request.headers['x-forwarded-for']` if [trustProxy](https://www.fastify.io/docs/master/Server/#trustproxy) option is enabled. Use it if you want to override this behavior
- `errorResponseBuilder`: a function to generate a custom response object. Defaults to `(request, context) => ({statusCode: 429, error: 'Too Many Requests', message: ``Rate limit exceeded, retry in ${context.after}``})`
- `addHeadersOnExceeding`: define which headers should be added in the response when the limit is not reached. Defaults all the headers will be shown
- `addHeaders`: define which headers should be added in the response when the limit is reached. Defaults all the headers will be shown
- `enableDraftSpec`: if `true` it will change the HTTP rate limit headers following the IEFT draft document. More information at [draft-ietf-httpapi-ratelimit-headers.md](https://github.com/ietf-wg-httpapi/ratelimit-headers/blob/f6a7bc7560a776ea96d800cf5ed3752d6d397b06/draft-ietf-httpapi-ratelimit-headers.md).

`keyGenerator` example usage:
```js
fastify.register(require('fastify-rate-limit'), {
  /* ... */
  keyGenerator: function (request) {
    return request.headers['x-real-ip'] // nginx
    || request.headers['x-client-ip'] // apache
    || request.headers['x-forwarded-for'] // use this only if you trust the header
    || request.session.username // you can limit based on any session value
    || request.ip // fallback to default
  }
})
```

Variable `max` example usage:
```js
// In the same timeWindow, the max value can change based on request and/or key like this
fastify.register(rateLimit, {
  /* ... */
  keyGenerator (request) { return request.headers['service-key'] },
  max: async (request, key) => { return key === 'pro' ? 3 : 2 },
  timeWindow: 1000
})
```

`errorResponseBuilder` example usage:
```js
fastify.register(require('fastify-rate-limit'), {
  /* ... */
  errorResponseBuilder: function (request, context) {
    return {
      code: 429,
      error: 'Too Many Requests',
      message: `I only allow ${context.max} requests per ${context.after} to this Website. Try again soon.`,
      date: Date.now(),
      expiresIn: context.ttl // milliseconds
    }
  }
})
```

Dynamic `allowList` example usage:
```js
fastify.register(require('fastify-rate-limit'), {
  /* ... */
  allowList: function (request, key) {
    return request.headers['x-app-client-id'] === 'internal-usage'
  }
})
```

Custom `hook` example usage (after authentication):
```js
fastify.register(require('fastify-rate-limit'), {
  hook: 'preHandler',
  keyGenerator: function (request) {
    return request.userId || request.ip
  }
})

fastify.decorateRequest('userId', '')
fastify.addHook('preHandler', async function (request) {
  const { userId } = request.query
  if (userId) {
    request.userId = userId
  }
})
```

Custom `store` example usage:

NOTE: The ```timeWindow``` will always be passed as the numeric value in millseconds into the store's constructor.

```js
function CustomStore (options) {
  this.options = options
  this.current = 0
}

CustomStore.prototype.incr = function (key, cb) {
  const timeWindow = this.options.timeWindow
  this.current++
  cb(null, { current: this.current, ttl: timeWindow - (this.current * 1000) })
}

CustomStore.prototype.child = function (routeOptions) {
  // We create a merged copy of the current parent parameters with the specific
  // route parameters and pass them into the child store.
  const childParams = Object.assign(this.options, routeOptions)
  const store = new CustomStore(childParams)
  // Here is where you may want to do some custom calls on the store with the information
  // in routeOptions first...
  // store.setSubKey(routeOptions.method + routeOptions.url)
  return store
}

fastify.register(require('fastify-rate-limit'), {
  /* ... */
  store: CustomStore
})
```

The `routeOptions` object passed to the `child` method of the store will contain the same options that are detailed above for plugin registration with any specific overrides provided on the route. In addition, the following parameter is provided:

- `routeInfo`: The configuration of the route including `method`, `url`, `path`, and the full route `config`

### Options on the endpoint itself

Rate limiting can be also can be configured at the route level, applying the configuration independently.

For example the `allowList` if configured:
 - on plugin registration will affect all endpoints within the encapsulation scope
 - on route declaration will affect only the targeted endpoint

The global allowlist is configured when registering it with `fastify.register(...)`.

The endpoint allowlist is set on the endpoint directly with the `{ config : { rateLimit : { allowList : [] } } }` object.

ACL checking is performed based on the value of the key from the `keyGenerator`.

In this example we are checking the IP address, but it could be an allowlist of specific user identifiers (like JWT or tokens):

```js
const fastify = require('fastify')()

fastify.register(require('fastify-rate-limit'),
  {
    global : false, // don't apply these settings to all the routes of the context
    max: 3000, // default global max rate limit
    allowList: ['192.168.0.10'], // global allowlist access.
    redis: redis, // custom connection to redis
  })

// add a limited route with this configuration plus the global one
fastify.get('/', {
  config: {
    rateLimit: {
      max: 3,
      timeWindow: '1 minute'
    }
  }
}, (request, reply) => {
  reply.send({ hello: 'from ... root' })
})

// add a limited route with this configuration plus the global one
fastify.get('/private', {
  config: {
    rateLimit: {
      max: 3,
      timeWindow: '1 minute'
    }
  }
}, (request, reply) => {
  reply.send({ hello: 'from ... private' })
})

// this route doesn't have any rate limit
fastify.get('/public', (request, reply) => {
  reply.send({ hello: 'from ... public' })
})

// add a limited route with this configuration plus the global one
fastify.get('/public/sub-rated-1', {
  config: {
    rateLimit: {
      timeWindow: '1 minute',
      allowList: ['127.0.0.1'],
      onExceeding: function (request) {
        console.log('callback on exceededing ... executed before response to client')
      },
      onExceeded: function (request) {
        console.log('callback on exceeded ... to black ip in security group for example, request is give as argument')
      }
    }
  }
}, (request, reply) => {
  reply.send({ hello: 'from sub-rated-1 ... using default max value ... ' })
})
```

In the route creation you can override the same settings of the plugin registration plus the following additional options:

- `onExceeding` : callback that will be executed each time a request is made to a route that is rate limited
- `onExceeded` : callback that will be executed when a user reached the maximum number of tries. Can be useful to blacklist clients

You may also want to set a global rate limiter and then disable on some routes:

```js
const fastify = require('fastify')()

fastify.register(require('fastify-rate-limit'), {
  max: 100,
  timeWindow: '1 minute'
})

// add a limited route with global config
fastify.get('/', (request, reply) => {
  reply.send({ hello: 'from ... rate limited root' })
})

// this route doesn't have any rate limit
fastify.get('/public', {
  config: {
    rateLimit: false
  }
}, (request, reply) => {
  reply.send({ hello: 'from ... public' })
})

// add a limited route with global config and different max
fastify.get('/private', {
  config: {
    rateLimit: {
      max: 9
    }
  }
}, (request, reply) => {
  reply.send({ hello: 'from ... private and more limited' })
})
```

### Examples of Custom Store

These examples show an overview of the `store` feature and you should take inspiration from it and tweak as you need:

- [Knex-SQLite](./example/example-knex.js)
- [Knex-MySQL](./example/example-knex-mysql.js)
- [Sequelize-PostgreSQL](./example/example-sequelize.js)

### IETF Draft Spec Headers

The response will have the following headers if `enableDraftSpec` is `true`:


| Header | Description |
|--------|-------------|
|`ratelimit-limit`       | how many requests the client can make
|`ratelimit-remaining`   | how many requests remain to the client in the timewindow
|`ratelimit-reset`       | how many seconds must pass before the rate limit resets
|`retry-after`           | contains the same value in time as `ratelimit-reset`

<a name="license"></a>
## License
**[MIT](https://github.com/fastify/fastify-rate-limit/blob/master/LICENSE)**<br>

Copyright © 2018 Tomas Della Vedova
