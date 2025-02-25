# esbuild

Example that sets up [esbuild](https://esbuild.github.io/getting-started/) to bundle source code before it is compiled and built into a Wasm bundle by the [JavaScript SDK](https://js-compute-reference-docs.edgecompute.app) for [Fastly Compute](https://www.fastly.com/documentation/guides/compute/javascript/).

The example source code is a JSX file and holds dependencies on `react` and `react-dom`. It demonstrates serialization of a React component into a stream, included in a synthesized response from a Compute application. 

```jsx
import * as React from 'react';
import * as Server from 'react-dom/server';

const Greet = () => <h1>Hello, world!</h1>;
return new Response(
  await Server.renderToReadableStream(<Greet />),
  {
    status: 200,
    headers: {
      'Content-Type': 'text/html',
    },
  },
);
```

## How it works

The example invokes esbuild in `package.json`'s `prebuild` script, so that it gets run before `build`.

```json
{
  "scripts": {
    "prebuild": "esbuild ./src/index.jsx --bundle --platform=neutral --outfile=./bin/index.js --conditions=fastly,edge-light --external:\"fastly:*\"",
    "build": "js-compute-runtime ./bin/index.js ./bin/main.wasm",
    "start": "fastly compute serve",
    "deploy": "fastly compute publish"
  }
}
```

The following is an explanation of the `prebuild` script:
- `esbuild` - The `esbuild` command
- `./src/index.jsx` - The entry point source file
- `--bundle` - Tells esbuild to run in bundle mode, in other words, to inline the dependency files into the output file
- `--platform=neutral` - Tells esbuild to use [neutral defaults](https://esbuild.github.io/api/#platform)
- `--outfile=./bin/index.js` - Tells esbuild where to place the output bundled file
- `--conditions=fastly,edge-light` - Adds `fastly` and `edge-light` to the list of conditions to consider when encountering modules that define [conditional exports](https://nodejs.org/api/packages.html#conditional-exports)
  - In particular, `react-dom/server` needs to be imported using the `edge-light` conditional export to work under Compute.
- `--external:"fastly:*"` - Adds an exception to `--bundle` so that imports that meet the specification `fastly:*` are not inlined into the bundle

Additionally, `build` is set up to source its input from `./bin/index.js`, which is the output of esbuild.

