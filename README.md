# Official Dub.co Typescript API Library

[![NPM version](https://img.shields.io/npm/v/dub.svg)](https://npmjs.org/package/dub)

This library provides convenient access to the Dub REST API from server-side TypeScript or JavaScript.

The REST API documentation can be found [on dub.co](https://dub.co/help). The full API of this library can be found in [api.md](https://www.github.com/dubinc/dub-node/blob/main/api.md).

## Installation

```sh
npm install --save dub
# or
yarn add dub
```

## Usage

The full API of this library can be found in [api.md](https://www.github.com/dubinc/dub-node/blob/main/api.md).

<!-- prettier-ignore -->
```js
import Dub from 'dub';

const dub = new Dub({
  token: process.env['DUB_API_KEY'], // This is the default and can be omitted
  projectSlug: 'dub_project_slug',
});

async function main() {
  const link = await dub.links.create({ url: 'google.com' });

  console.log(link);
}

main();
```

### Request & Response types

This library includes TypeScript definitions for all request params and response fields. You may import and use them like so:

<!-- prettier-ignore -->
```ts
import Dub from 'dub';

const dub = new Dub({
  token: process.env['DUB_API_KEY'], // This is the default and can be omitted
  projectSlug: 'dub_project_slug',
});

async function main() {
  const params: Dub.LinkCreateParams = { url: 'google.com' };
  const link: Dub.Link = await dub.links.create(params);

  console.log(link);
}

main();
```

Documentation for each method, request param, and response field are available in docstrings and will appear on hover in most modern editors.

## Handling errors

When the library is unable to connect to the API,
or if the API returns a non-success status code (i.e., 4xx or 5xx response),
a subclass of `APIError` will be thrown:

<!-- prettier-ignore -->
```ts
async function main() {
  const link = await dub.links
    .create({ url: 'google.com' })
    .catch((err) => {
      if (err instanceof Dub.APIError) {
        console.log(err.status); // 400
        console.log(err.name); // BadRequestError
        console.log(err.headers); // {server: 'nginx', ...}
      } else {
        throw err;
      }
    });
}

main();
```

Error codes are as followed:

| Status Code | Error Type                 |
| ----------- | -------------------------- |
| 400         | `BadRequestError`          |
| 401         | `AuthenticationError`      |
| 403         | `PermissionDeniedError`    |
| 404         | `NotFoundError`            |
| 422         | `UnprocessableEntityError` |
| 429         | `RateLimitError`           |
| >=500       | `InternalServerError`      |
| N/A         | `APIConnectionError`       |

### Retries

Certain errors will be automatically retried 2 times by default, with a short exponential backoff.
Connection errors (for example, due to a network connectivity problem), 408 Request Timeout, 409 Conflict,
429 Rate Limit, and >=500 Internal errors will all be retried by default.

You can use the `maxRetries` option to configure or disable this:

<!-- prettier-ignore -->
```js
// Configure the default for all requests:
const dub = new Dub({
  maxRetries: 0, // default is 2
  projectSlug: 'dub_project_slug',
});

// Or, configure per-request:
await dub.links.create({ url: 'google.com' }, {
  maxRetries: 5,
});
```

### Timeouts

Requests time out after 1 minute by default. You can configure this with a `timeout` option:

<!-- prettier-ignore -->
```ts
// Configure the default for all requests:
const dub = new Dub({
  timeout: 20 * 1000, // 20 seconds (default is 1 minute)
  projectSlug: 'dub_project_slug',
});

// Override per-request:
await dub.links.create({ url: 'google.com' }, {
  timeout: 5 * 1000,
});
```

On timeout, an `APIConnectionTimeoutError` is thrown.

Note that requests which time out will be [retried twice by default](#retries).

## Advanced Usage

### Accessing raw Response data (e.g., headers)

The "raw" `Response` returned by `fetch()` can be accessed through the `.asResponse()` method on the `APIPromise` type that all methods return.

You can also use the `.withResponse()` method to get the raw `Response` along with the parsed data.

<!-- prettier-ignore -->
```ts
const dub = new Dub();

const response = await dub.links
  .create({ url: 'google.com' })
  .asResponse();
console.log(response.headers.get('X-My-Header'));
console.log(response.statusText); // access the underlying Response object

const { data: link, response: raw } = await dub.links
  .create({ url: 'google.com' })
  .withResponse();
console.log(raw.headers.get('X-My-Header'));
console.log(link.id);
```

## Customizing the fetch client

By default, this library uses `node-fetch` in Node, and expects a global `fetch` function in other environments.

If you would prefer to use a global, web-standards-compliant `fetch` function even in a Node environment,
(for example, if you are running Node with `--experimental-fetch` or using NextJS which polyfills with `undici`),
add the following import before your first import `from "Dub"`:

```ts
// Tell TypeScript and the package to use the global web fetch instead of node-fetch.
// Note, despite the name, this does not add any polyfills, but expects them to be provided if needed.
import 'dub/shims/web';
import Dub from 'dub';
```

To do the inverse, add `import "dub/shims/node"` (which does import polyfills).
This can also be useful if you are getting the wrong TypeScript types for `Response` - more details [here](https://github.com/dubinc/dub-node/tree/main/src/_shims#readme).

You may also provide a custom `fetch` function when instantiating the client,
which can be used to inspect or alter the `Request` or `Response` before/after each request:

```ts
import { fetch } from 'undici'; // as one example
import Dub from 'dub';

const client = new Dub({
  fetch: async (url: RequestInfo, init?: RequestInfo): Promise<Response> => {
    console.log('About to make a request', url, init);
    const response = await fetch(url, init);
    console.log('Got response', response);
    return response;
  },
});
```

Note that if given a `DEBUG=true` environment variable, this library will log all requests and responses automatically.
This is intended for debugging purposes only and may change in the future without notice.

## Configuring an HTTP(S) Agent (e.g., for proxies)

By default, this library uses a stable agent for all http/https requests to reuse TCP connections, eliminating many TCP & TLS handshakes and shaving around 100ms off most requests.

If you would like to disable or customize this behavior, for example to use the API behind a proxy, you can pass an `httpAgent` which is used for all requests (be they http or https), for example:

<!-- prettier-ignore -->
```ts
import http from 'http';
import HttpsProxyAgent from 'https-proxy-agent';

// Configure the default for all requests:
const dub = new Dub({
  httpAgent: new HttpsProxyAgent(process.env.PROXY_URL),
  projectSlug: 'dub_project_slug',
});

// Override per-request:
await dub.links.create({ url: 'google.com' }, {
  baseURL: 'http://localhost:8080/test-api',
  httpAgent: new http.Agent({ keepAlive: false }),
})
```

## Semantic Versioning

This package generally follows [SemVer](https://semver.org/spec/v2.0.0.html) conventions, though certain backwards-incompatible changes may be released as minor versions:

1. Changes that only affect static types, without breaking runtime behavior.
2. Changes to library internals which are technically public but not intended or documented for external use. _(Please open a GitHub issue to let us know if you are relying on such internals)_.
3. Changes that we do not expect to impact the vast majority of users in practice.

We take backwards-compatibility seriously and work hard to ensure you can rely on a smooth upgrade experience.

We are keen for your feedback; please open an [issue](https://www.github.com/dubinc/dub-node/issues) with questions, bugs, or suggestions.

## Requirements

TypeScript >= 4.5 is supported.

The following runtimes are supported:

- Node.js 18 LTS or later ([non-EOL](https://endoflife.date/nodejs)) versions.
- Deno v1.28.0 or higher, using `import Dub from "npm:dub"`.
- Bun 1.0 or later.
- Cloudflare Workers.
- Vercel Edge Runtime.
- Jest 28 or greater with the `"node"` environment (`"jsdom"` is not supported at this time).
- Nitro v2.6 or greater.

Note that React Native is not supported at this time.

If you are interested in other runtime environments, please open or upvote an issue on GitHub.
