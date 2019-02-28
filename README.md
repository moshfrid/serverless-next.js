# Serverless Nextjs Plugin

[![Build Status](https://travis-ci.org/danielcondemarin/serverless-nextjs-plugin.svg?branch=master)](https://travis-ci.org/danielcondemarin/serverless-nextjs-plugin)
[![Coverage Status](https://coveralls.io/repos/github/danielcondemarin/serverless-nextjs-plugin/badge.svg?branch=master)](https://coveralls.io/github/danielcondemarin/serverless-nextjs-plugin?branch=master)

A [serverless framework](https://serverless.com/) plugin to deploy nextjs apps.

The plugin targets Next-8 serverless mode. See https://nextjs.org/blog/next-8/#serverless-nextjs.

## Motivation

Next 8 released [official support](https://nextjs.org/blog/next-8/#serverless-nextjs) for serverless! It doesn't work out of the box with AWS Lambdas, instead, next provides a low level API which this plugin uses to deploy the serverless pages.

Nextjs serverless page handler signature:

`exports.render = function(req, res) => {...}`

AWS Lambda handler:

`exports.handler = function(event, context, callback) {...}`

The plugin adds a compat layer between the nextjs page bundles and AWS Lambda at build time:

```
const page = require(".next/serverless/pages/somePage.js");

module.exports.render = (event, context, callback) => {
  const { req, res } = compatLayer(event, callback);
  page.render(req, res);
};
```

## Getting started

Let's first configure the `serverless.yml` file:

### Plugin configuration

The plugin only needs to know where your next.config.js file is located. Note it expects the directory and not the actual file path. E.g. `./nextApp` where inside nextApp there is `next.config.js`.

```
custom:
  serverless-nextjs:
    nextConfigDir: '/dir/to/my/nextApp'
```

### Page functions

Configure the functions for the next serverless pages as you would do for any other serverless function.

```
functions:
  home-page:
    handler: build/serverless/pages/home.render
    events:
      - http:
          path: home
          method: get
  about-page:
    handler: build/serverless/pages/about.render
    events:
      - http:
          path: about
          method: get

package:
  # exclude everything, except for the page bundles
  exclude:
    - ./**/*
  include:
    # important to use {handler}.* as there will be 2 files per function handler
    - build/serverless/pages/home.*
    - build/serverless/pages/about.*
```

Since the next page bundles are self contained, you can exclude everything. However, make sure when you are including the page bundles, to use the pattern _build/serverless/pages/{pageName}.\*_. This makes sure the compat files created by the plugin are also included in the deploy artifact.

### Next configuration

In your `next.config.js` make sure the configuration is set like:

```
module.exports = {
  target: "serverless",
  distDir: "build",
  assetPrefix: "https://s3.amazonaws.com/your-bucket-name"
}
```

`target: serverless`

This is a requirement for the plugin to work. When next has the target set to serverless, it will compile serverless page bundles.

`distDir: build`

Make sure you don't use the default value `.next` as it seems to break the Lambda deployment, probably because is a dot directory. `build` is fine, but could be any other name.

`assetPrefix: "https://s3.amazonaws.com/your-bucket-name"`

Any valid bucket URL will work, e.g. "https://your-bucket-name.s3.amazonaws.com/".

The plugin will parse the bucket name from the `assetPrefix` and will create a new public S3 bucket using the parsed bucket name. The first time the serverless stack is provisioned, it is assumed there isn't a bucket with this name already. Do _note that bucket names must be unique globally_. On deployment, the plugin will upload the next static assets to the bucket.

After you've configured the above, simply run:

`serverless deploy`

Visit the API GW endpoints and the next pages should be working 🎉

## Examples

See the `examples/` directory.

## Roadmap

- Serverless functions created at build time, so users don't have to manually specify them in the `serverless.yml`.
- Look into mitigating cold starts, maybe just add an example using another plugin which solves this.
- More examples.

## Note

This is still a WIP so is quite likely there will be breaking changes.

Any feedback is really appreciated. Also PRs are welcome :)
