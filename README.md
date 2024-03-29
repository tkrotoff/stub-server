# @tkrotoff/stub-server

[![npm version](https://badge.fury.io/js/%40tkrotoff%2Fstub-server.svg)](https://www.npmjs.com/package/@tkrotoff/stub-server)
[![Node.js CI](https://github.com/tkrotoff/stub-server/workflows/Node.js%20CI/badge.svg?branch=master)](https://github.com/tkrotoff/stub-server/actions)
[![Bundle size](https://badgen.net/bundlephobia/minzip/@tkrotoff/stub-server)](https://bundlephobia.com/result?p=@tkrotoff/stub-server)
[![Prettier](https://img.shields.io/badge/code_style-prettier-ff69b4.svg)](https://github.com/prettier/prettier)
[![Airbnb Code Style](https://badgen.net/badge/code%20style/airbnb/ff5a5f?icon=airbnb)](https://github.com/airbnb/javascript)

Stub/mock server for REST APIs

For each route, decide what will happen: a JSON stub, a piece of JS or redirect to a real server

- Use it with Express, [webpack-dev-server](https://github.com/webpack/webpack-dev-server), Next.js or the command line
- Supports stubs written in JSON, JS, TypeScript, HTML, jpg...
- Can redirect requests to another host thx to [node-http-proxy](https://github.com/http-party/node-http-proxy)
- No need to restart stub-server if you modify a stub
- The HTTP status code returned is determined from the stub filename: \*\_200\_\*.json, \*\_500\_\*.html...
- Configurable delays to simulate slow APIs (helps find bugs and possible improvements to your app - add a spinner, disable a submit button...)
- Access to request & response objects (allows to change headers, request URL...)

## Usage

`npm install --save-dev @tkrotoff/stub-server`

Example: https://github.com/tkrotoff/MarvelHeroes

### Proposed file organization

```
stubs/routes/my_api1_GET_200_OK.json
             my_api2_GET_200_OK.jpg
             my_api3_POST_400_BadRequest-invalidField.ts
             my_api4_POST_400_BadRequest-invalidField.js
             my_api5_DELETE_500_InternalServerError.html
             my_api7_GET_200_OK.json
             my_api7_POST_200_OK.json
             my_api8_GET.ts
             my_api9_GET.js
             my_api10_GET_200_OK-param1.json
             my_api10_GET_200_OK-param2.json
stubs/config.ts
webpack.config.ts
```

### stubs/config.ts

```TypeScript
import path from 'path';
import { StubServerConfig } from '@tkrotoff/stub-server';

const prod = 'https://myapp.com';

const stubsPath = path.resolve(__dirname, 'routes');

const config: StubServerConfig = {
  delay: { min: 500, max: 3000 },
  routes: {
    '/my/api1': { GET: `${stubsPath}/my_api1_GET_200_OK.json` },
    '/my/api2': { GET: `${stubsPath}/my_api2_GET_200_OK.jpg` },
    '/my/api3': { POST: `${stubsPath}/my_api3_POST_400_BadRequest-invalidField.ts` },
    '/my/api4': { POST: `${stubsPath}/my_api4_POST_400_BadRequest-invalidField.js` },
    '/my/api5': { DELETE: `${stubsPath}/my_api5_DELETE_500_InternalServerError.html` },
    '/my/api6/:id': {
      // Here stub-server works as a proxy,
      // example of request sent: PUT https://myapp.com/my/api6/132379
      PUT: prod
    },
    '/my/api7': {
      delay: { min: 1000, max: 1000 },
      GET: `${stubsPath}/my_api7_GET_200_OK.json`,
      POST: {
        delay: { min: 0, max: 0 },
        response: req => {
          req.headers.origin = prod;
          return `${stubsPath}/my_api7_POST_200_OK.json`;
        }
      }
    },
    '/my/api8': { GET: `${stubsPath}/my_api8_GET.ts`},
    '/my/api9': { GET: `${stubsPath}/my_api9_GET.js`},
    '/my/api10/:id': { GET: req => `${stubsPath}/my_api10_GET_200_OK-${req.params.id}.json` }
  }
};

const rootApiPath = 'https://myapp.com/client/:clientApi';
config.routes[`${rootApiPath}/my/api7`] = { GET: `${stubsPath}/my_api7_GET_200_OK.json` };

export default config; // Or "exports.default = config"
```

Note: `stubs/config.ts` written in TypeScript instead of JavaScript requires [ts-node](https://github.com/TypeStrong/ts-node)

### webpack.config.ts

Configuration with [webpack-dev-server](https://github.com/webpack/webpack-dev-server)

```TypeScript
import { stubServer } from '@tkrotoff/stub-server';

...

  devServer: {
    // With webpack-dev-server >= v4.7.0
    setupMiddlewares: (middlewares, devServer) => {
      const configPath = path.resolve(__dirname, 'stubs/config');
      stubServer(configPath, devServer.app!);
      return middlewares;
    }

    // With webpack-dev-server from v4.0.0 to v4.6.0
    onBeforeSetupMiddleware: (devServer) => {
      const configPath = path.resolve(__dirname, 'stubs/config');
      stubServer(configPath, devServer.app!);
    }

    // With webpack-dev-server < v4.0.0
    before: app => {
      const configPath = path.resolve(__dirname, 'stubs/config');
      stubServer(configPath, app);
    }

    ...
  },
```

### stubs/routes/my_api1_GET_200_OK.json

```JSON
{
  "foo": "bar"
}
```

### stubs/routes/my_api3_POST_400_BadRequest-invalidField.ts

```TypeScript
export default {
  error: 'Invalid field'
};
```

### stubs/routes/my_api4_POST_400_BadRequest-invalidField.js

```JavaScript
module.exports = {
  error: 'Invalid field'
};
```

### stubs/routes/my_api8_GET.ts

```TypeScript
import express from 'express';

export default function stub(req: express.Request, res: express.Response) {
  res.send('Hello, World!');
}
```

### stubs/routes/my_api9_GET.js

```JavaScript
module.exports = (req, res) => {
  res.send('Hello, World!');
};
```

## Command line

```
Usage: stub-server [options] <config>

Arguments:
  config         path to the config file

Options:
  --port <port>  stub server port (default: "12345")
  --no-delay     ignore any delay specified in the config
  -h, --help     display help for command
```

## Next.js

```JavaScript
// stubServer.js

const { stubServer } = require('@tkrotoff/stub-server');
const cors = require('cors');
const express = require('express');
const next = require('next');
const Log = require('next/dist/build/output/log');
const path = require('path');

const app = next({ dev: process.env.NODE_ENV !== 'production' });
const nextjsHandler = app.getRequestHandler();

const configPath = path.resolve(__dirname, 'stubs/config');
const port = 3000;

app.prepare().then(() => {
  const server = express();
  server.use(express.json());
  server.use(cors());

  stubServer(configPath, server);

  // Next.js only processes GET requests unless you are using [API Routes](https://nextjs.org/docs/api-routes/introduction)
  server.get('*', (req, res) => nextjsHandler(req, res));

  server.all('*', req => {
    throw new Error(`'${req.method} ${req.url}' should be declared in stubs/config.ts`);
  });

  server.listen(port, () => {
    Log.ready(`ready on port ${port}`);
  });
});
```

```JavaScript
// package.json

  "scripts": {
    "dev": "node stubServer.js", // Instead of "next dev"
    "build": "next build",
    "start": "next start"
    ...
  },
```

## How to configure stub-server both for stubs and as a proxy

To "connect" your app to an [environment](https://en.wikipedia.org/wiki/Deployment_environment) (prod, staging, integration...), you can use stub-server as a proxy (intermediary between your app requesting a resource and the server providing that resource).

Why? To configure delays for the HTTP requests (helps find bugs and possible improvements to your app), access to request & response objects...

```JavaScript
// package.json

  "scripts": {
    "start": "...",
    "start:staging": "ENVIRONMENT_NAME=staging npm run start",
    "start:prod": "ENVIRONMENT_NAME=prod npm run start"
    ...
  },
```

```TypeScript
// stubs/config.ts

import path from 'path';
import express from 'express';
import { GetStubFilenameOrUrl, StubServerConfig } from '@tkrotoff/stub-server';

const stubsPath = path.resolve(__dirname, 'routes');

type Environments = { [name: string]: { target: string; origin: string } | undefined };

const environments: Environments = {
  staging: {
    target: 'https://api.staging.myapp.com',
    origin: 'https://staging.myapp.com'
  },
  prod: {
    target: 'https://api.myapp.com',
    origin: 'https://myapp.com'
  }
};

const { target, origin } =
  environments[process.env.ENVIRONMENT_NAME ?? 'No environment specified'] ?? {};

function configureRequest(stubOrUrl: string): GetStubFilenameOrUrl {
  return (req: express.Request) => {
    // You can rewrite the request URL if needed
    req.url = req.url.replace('/prefix', '');

    if (origin !== undefined) {
      // May be required for some APIs
      req.headers.origin = origin;
    }

    return stubOrUrl;
  };
}

const config: StubServerConfig = {
  delay: { min: 500, max: 3000 },
  routes: {
    '/prefix/my/api1': {
      GET: configureRequest(
        // Here stub-server works as a proxy,
        // example of request sent: GET https://api.staging.myapp.com/my/api1
        target ??
        // ...or use a stub if no target specified
        `${stubsPath}/my_api1_GET_200_OK.json`
      )
    },
    ...
  }
};

export default config;
```

## Output generated

Examples of output (stdout) for requests processed by stub-server:

```
GET /account/payment-types?imagesWithUrl=true => stubs/routes/payment-types_200_OK.ts, delay: 442 ms
GET /account/captcha/1662973203224 => https://api.myapp.com, delay: 0 ms
GET /account/captcha/1663061235576 => stubs/routes/captcha_200_OK-242674.jpg, delay: 4 ms
POST /account/create => stubs/routes/account_create_201_Created-no-appro.json, delay: 169 ms
POST /auth/session?withNotification=true => https://api.myapp.com, delay: 34 ms
```

## Errors

If the stub is missing or the target is unknown, stub-server returns a 500 (Internal Server Error) response with the error in HTML format and also displays the error in the console.

Examples of errors in the console:

- `ENOENT: no such file or directory, open 'stubs/routes/captcha_200_OK-242674.jpg'`
- `Error: getaddrinfo ENOTFOUND api.myapp.com`

If you request an unknown route, stub-server does not process it and therefore won't display anything in the console. It's up to you to handle this case with a catch-all route:

```TypeScript
function missingStubHandler(req: express.Request) {
  throw new Error(`Missing stub for '${req.method} ${req.url}'`);
}

...

'/prefix/*': {
  GET: missingStubHandler,
  POST: missingStubHandler,
  PUT: missingStubHandler,
  PATCH: missingStubHandler,
  DELETE: missingStubHandler
}
```
