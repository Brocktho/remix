---
title: Gotchas
---

# Gotchas

As we've built Remix, we've been laser focused on production results and scalability for your users and team working in it. Because of this, some developer-experience and ecosystem-compatibility issues exist that we haven't smoothed over yet.

This document should help you get over these bumps.

## Server Code in Client Bundles

You may run into this strange error in the browser. It almost always means that server code made it into browser bundles.

```
TypeError: Cannot read properties of undefined (reading 'root')
```

For example, you can't import "fs-extra" directly into a route module:

```tsx bad filename=app/routes/index.tsx lines=[2] nocopy
import { json } from "@remix-run/node"; // or cloudflare/deno
import fs from "fs-extra";

export async function loader() {
  return json(await fs.pathExists("../some/path"));
}

export default function SomeRoute() {
  // ...
}
```

To fix it, move the import into a different module named `*.server.js` or `*.server.ts` and import from there. In our example here, we create a new file at `utils/fs-extra.server.js`:

```ts filename=app/utils/fs-extra.server.ts
export { default } from "fs-extra";
```

And then change our import in the route to the new "wrapper" module:

```tsx filename=app/routes/index.tsx lines=[3]
import { json } from "@remix-run/node"; // or cloudflare/deno

import fs from "~/utils/fs-extra.server";

export async function loader() {
  return json(await fs.pathExists("../some/path"));
}

export default function SomeRoute() {
  // ...
}
```

Even better, send a PR to the project to add `"sideEffects": false` to their package.json so that bundlers that tree shake know they can safely remove the code from browser bundles.

Similarly, you may run into the same error if you call a function at the top-level scope of your route module that depends on server-only code.

For example, [Remix upload handlers like `unstable_createFileUploadHandler` and `unstable_createMemoryUploadHandler`][remix-upload-handlers-like-unstable-create-file-upload-handler-and-unstable-create-memory-upload-handler] use Node globals under the hood and should only be called on the server. You can call either of these functions in a `*.server.js` or `*.server.ts` file, or you can move them into your route's `action` or `loader` function.

So instead of doing:

```tsx bad filename=app/routes/some-route.tsx lines=[3-6]
import { unstable_createFileUploadHandler } from "@remix-run/node"; // or cloudflare/deno

const uploadHandler = unstable_createFileUploadHandler({
  maxPartSize: 5_000_000,
  file: ({ filename }) => filename,
});

export async function action() {
  // use `uploadHandler` here ...
}
```

You should be doing:

```tsx filename=app/routes/some-route.tsx good lines=[4-7]
import { unstable_createFileUploadHandler } from "@remix-run/node"; // or cloudflare/deno

export async function action() {
  const uploadHandler = unstable_createFileUploadHandler({
    maxPartSize: 5_000_000,
    file: ({ filename }) => filename,
  });

  // use `uploadHandler` here ...
}
```

> Why does this happen?

Remix uses "tree shaking" to remove server code from browser bundles. Anything inside of Route module `loader`, `action`, and `headers` exports will be removed. It's a great approach but suffers from ecosystem compatibility.

When you import a third-party module, Remix checks the `package.json` of that package for `"sideEffects": false`. If that is configured, Remix knows it can safely remove the code from the client bundles. Without it, the imports remain because code may depend on the module's side effects (like setting global polyfills, etc.).

## Importing ESM Packages

You may try importing an ESM-only package into your app and see an error like this when server rendering:

```
Error [ERR_REQUIRE_ESM]: require() of ES Module /app/node_modules/dot-prop/index.js from /app/project/build/index.js not supported.
Instead change the require of /app/project/node_modules/dot-prop/index.js in /app/project/build/index.js to a dynamic import() which is available in all CommonJS modules.
```

To fix it, add the ESM package to the `serverDependenciesToBundle` option in your `remix.config.js` file.

In our case here, we're using the `dot-prop` package, so we would do it like this:

```js filename=remix.config.js
/** @type {import('@remix-run/dev').AppConfig} */
module.exports = {
  serverDependenciesToBundle: ["dot-prop"],
  // ...
};
```

> Why does this happen?

Remix compiles your server build to CJS and doesn't bundle your node modules. CJS modules can't import ESM modules.

Adding packages to `serverDependenciesToBundle` tells Remix to bundle the ESM module directly into the server build instead of requiring it at runtime.

> Isn't ESM the future?

Yes! Our plan is to allow you to compile your app to ESM on the server. However, that will come with the reverse problem of not being able to import some CommonJS modules that are incompatible with being imported from ESM! So even when we get there, we may still need this configuration.

You may ask why we don't just bundle everything for the server. We could, but that will slow down builds and make production stack traces all point to a single file for your entire app. We don't want to do that. We know we can smooth this over eventually without making that tradeoff.

With major deployment platforms now supporting ESM server side, we're confident the future is brighter than the past here. We're still working on a solid dev experience for ESM server builds, our current approach relies on some things that you can't do in ESM. We'll get there.

## `typeof window` checks

Because the same JavaScript code can run in the browser as well as the server, sometimes you need to have a part of your code that only runs in one context or the other:

```ts bad
if (typeof window === "undefined") {
  // running in a server environment
} else {
  // running in a browser environment
}
```

This works fine in a Node.js environment, however, Deno actually supports `window`! So if you really want to check whether you're running in the browser, it's better to check for `document` instead:

```ts good
if (typeof document === "undefined") {
  // running in a server environment
} else {
  // running in a browser environment
}
```

This will work for all JS environments (Node.js, Deno, Workers, etc.).

[esbuild]: https://esbuild.github.io/

## Browser extensions injecting code

You may run into this warning in the browser:

```
Warning: Did not expect server HTML to contain a <script> in <html>.
```

This is a hydration warning from React, and is most likely due to one of your browser extensions injecting scripts into the server-rendered HTML, creating a difference with the resulting HTML.

Check out the page in incognito mode, the warning should disappear.

## CSS bundle being incorrectly tree-shaken

When using [CSS bundling features][css-bundling] in combination with `export *` (e.g. when using an index file like `components/index.ts` that re-exports from all sub-directories) you may find that styles from the re-exported modules are missing from the build output.

This is due to an [issue with esbuild's CSS tree shaking][esbuild-css-tree-shaking-issue]. As a workaround, you should use named re-exports instead.

```diff
-export * from "./Button";
+export { Button } from "./Button";
```

Note that, even if this issue didn't exist, we'd still recommend using named re-exports! While it may introduce a bit more boilerplate, you get explicit control over the module's public interface rather than inadvertently exposing everything.

[remix-upload-handlers-like-unstable-create-file-upload-handler-and-unstable-create-memory-upload-handler]: ../utils/parse-multipart-form-data#uploadhandler
[css-bundling]: ../guides/styling#css-bundling
[esbuild-css-tree-shaking-issue]: https://github.com/evanw/esbuild/issues/1370


## Minimizing Bundle Size

As you continue to develop you may start including larger packages and notice an impact on your build time. There are a few things you can do to minimize your build times that'll also end up being great for your end user (less javascript, faster load times!). Often times larger packages will centralize a location to easily import components or hooks into your app, this is great for convenience but can make it difficult for the compiler to tree shake out code that isn't used.

Take for example MUI:

```tsx bad
import { Button, TextField } from '@mui/material';
```

Here Button and TextField are imported from the top-level index of the MUI package, this won't have a massive effect on users but can create more work for the compiler giving you longer build times. Instead you can do the following:

```tsx good
import Button from "@mui/material/Button";
import TextField from "@mui/material/TextField";
```

This lessens the work the compiler has to do and guarantees a smaller overall bundle size.

Sometimes you may also import packages that you intend for server use only. Sometimes this will cause immediate errors, but other times the compiler may try adding polyfills that increase your overall bundle size. Take for example core node modules like node:crypto.

```tsx bad
import crypto from "node:crypto";

import { json } from "@remix-run/node"; 

export async function loader() {
  const uuid = crypto.randomUUID();
  return json({uuid});
}
```

Importing crypto this way could lead to the bundler including polyfills for crypto into your client build even if it wasn't directly used on the client. An easy way to get around this is to centralize server imports inside a .server file. For example:

```tsx filename=app/utils/Node.server.ts
import crypto from "node:crypto";
import fs from "node:fs";

export { fs, crypto };
```

```tsx good filename=app/routes/some/route.tsx
import { json } from "@remix-run/node";

import { crypto } from "~/utils/Node.server";

export async function loader(){
  const uuid = crypto.randomUUID();
  return json({uuid});
}
```

This simple change can make a large change in your application build times and even help your end users load your application that much faster!