# LavaMoat - a Browserify Plugin for creating LavaMoat-protected builds

**NOTE: under rapid develop, not ready for production use, has not been audited, etc**

`lavamoat-browserify` is a [browserify][BrowserifyGithub] plugin for generating app bundles protected by [LavaMoat](https://github.com/LavaMoat/overview), where modules are defined in [SES][SesGithub] containers. It aims to reduce the risk of "software supplychain attacks", malicious code in the app dependency graph.

[BrowserifyGithub]: https://github.com/browserify/browserify
[SesGithub]: https://github.com/agoric/SES

```
npm i lavamoat-browserify
```

## Anatomy of a LavaMoat bundle

The `lavamoat-browserify` plugin replaces the last internal build step of the browserify [compiler pipeline](https://github.com/browserify/browserify-handbook#compiler-pipeline). This step takes all the modules and their metadata and outputs the final bundle content, including the kernel and LavaMoat configuration file.

LavaMoat builds differ from standard browserify builds in that they:

1. Include the app-specified LavaMoat configuration

This tells the kernel what execution environment each module should be instantiated with, and what other modules may be brought in as dependencies.

2. Use a custom LavaMoat kernel

This kernel enforces the LavaMoat config. When requested, a module is initialized, usually by evaluation inside a SES container. The kernel also protects the module's exports from modification via a strategy provided in the config such as SES hardening, deep copies, or copy-on-write views.

3. Bundle the module sources as strings

Modules are SES eval'd with access only to the platform APIs specificied in the config.

The result is a bundle that should work just as before, but provides some protection against supplychain attacks.

## CLI Usage
See below: JS Usage

Ensure you have browserify installed:

```
npm i -g browserify
```

Base LavaMoat command syntax:

```
browserify index.js --plugin [ lavamoat-browserify --writeAutoConfig --config —-configOverride ] 
```

* `index.js` is the typical entry point for Browserify.
* `--plugin` tells Browserify to include the following plugin in the bundle process.
* `lavamoat-browserify` is the plugin name, which accepts 3 different arguments. All arguments are optional:
    * `--writeAutoConfig`: `true` tells the plugin to automatically generate a configuration file. Otherwise, pass in `false` or omit the option completely.
    * `--config`: Specifies a pre-set configuration file. Can be passed in as a direct config object (type `object`), the path to the existing config file (type `string`) or a function which returns which returns said object or file path.

        * When used with `writeAutoConfig`, it must be the file path to be used for the auto-generated config. `writeAutoConfig` & `config` should otherwise be mutually exclusive.

    * `--configOverride`: Specifies another configuration, if any, to be merged with `config`. `configOverride` takes overwrite precedence; key `A` and its value on `configOverride` will overwrite key `A` and its value on `config`, so use with caution. Can be passed in as an `object`, `string`, or `function` in the same sense as `config` shown above.

* IMPORTANT: Be mindful of the spacings at the beginning and end of the brackets `[ ... ]`, Browserify will fail without them. 


### Examples 

#### Automated Config Generation

Generate a `lavamoat-config.json` file:

```
browserify index.js --plugin [ lavamoat-browserify --writeAutoConfig ] > /dev/null
```

This command runs Browserify with LavaMoat and the `--writeAutoConfig` flag. LavaMoat automatically parses your dependencies and outputs a config with default settings at the default path `./lavamoat/lavamoat-config.json` unless otherwise specified by `--config`.

Be sure to use the same Browserify configuration (eg. plugins and transforms like babelify) that you normally use, so that it can parse the code as it will appear in your final bundle. Ignore the output of this command.

`writeAutoConfig` also generates a default override config file, `lavamoat-config-override.json`, in the same default directory, if it doesn't already exist. This file is useful for applying custom modifications to the config that won’t be over written. See the config override section below for more details.

#### Building

Once you have a config (generated from `writeAutoConfig` or otherwise), create the Browserify bundle with LavaMoat, including a reference to the config and pipe it to a bundle file, `bundle.js` in this example.

Config as a filepath (`string`):

```
browserify index.js --plugin [ lavamoat-browserify --config ./lavamoat/lavamoat-config.json ] > bundle.js
```

Config as a standalone object (`object`) (not recommended):

```
browserify index.js --plugin [ lavamoat-browserify --config "{ resources: { <root>: { packages: { react: true } } } }" ] > bundle.js
```

Config as a function, must return a filepath or a standalone object:

```
browserify index.js --plugin [ lavamoat-browserify --config "() => './lavamoat/lavamoat-config.json'" ] > bundle.js
```

#### Config Override

You may wish to modify the config to suit your needs.

WARNING: Do not edit the autogenerated config `lavamoat-config.json` directly. It will be overwritten if a new bundle is created using LavaMoat. Instead, edit the `lavamoat-config-overwrite.json` file generated upon running LavaMoat with `writeAutoConfig`. It merges with the original config, always taking overwrite precedence. 

You may also create and specify your own override config. Ensure it is placed within `./lavamoat/` relative to the current working directory.

Specify an override config. `overrideConfig` can be a file path (`string`), standalone object (`object`), or a function that returns a the filepath or standalone object.

`configOverride` as a file path string:

```
browserify index.js --plugin [ lavamoat-browserify —config ./lavamoat/lavamoat-config.json —-configOverride ./lavamoat/lavamoat-config-override.json ] > bundle.js 
```

For configOverride as an `object` or a `function`, follow the same process as `config` above.

LavaMoat automatically applies an override config if it’s present in `./lavamoat/lavamoat-config-override.json` (relative to the current working directory) but not specified in the bundle command:

```
browserify index.js --plugin [ lavamoat-browserify —config ./lavamoat/lavamoat-config.json ] > bundle.js
```

## JS Usage

You may initiate a Browserify build with LavaMoat directly from the Browserify API. Every plugin option for LavaMoat outlined in the CLI usage is the same for JS. Relevant details are outlined in the CLI section, but omitted here to avoid duplication. 

### Examples 

#### Automated Config Generation

Generate `lavamoat-config.json` & `lavamoat-config-override.json` files:

```javascript
const browserify = require('browserify')

const lavamoatOpts = {
    writeAutoConfig: true
}

const bundler = browserify(['./index.js'], {
  plugin: [
    ['lavamoat-browserify', lavamoatOpts]
  ]
})
```

#### Building

Once you have a config (generated from `writeAutoConfig` or otherwise), create the Browserify bundle with LavaMoat, including a reference to the config and pipe it to a bundle file, `bundle.js` in this example.

Config as a filepath (`string`):

```javascript
const browserify = require('browserify')
const fs = require('fs')

const lavamoatOpts = {
    config: './lavamoat/lavamoat-config.json'
}

const bundler = browserify(['./index.js'], {
  plugin: [
    ['lavamoat-browserify', lavamoatOpts]
  ]
})

bundler.bundle()
  .pipe(fs.createWriteStream('./bundle.js'))
```

Config as a standalone object (`object`):

```javascript
const lavamoatOpts = {
    config: {
        resources: {
             <root>: {
                 packages: {
                     react: true
                }
            }
        }
    }
}
```

Config as a function, must return a filepath or a standalone object:

```javascript
const lavamoatOpts = {
    config: () => "./lavamoat/lavamoat-config.json"
}
```

OR

```javascript
const configObject = {
    resources: {
        <root>: {
            packages: {
                react: true
            }
        }
    }
}
const lavamoatOpts = {
    config: () => configObject
}
```

#### Config Override

Specify an override config. `overrideConfig` can be a file path (`string`), standalone object (`object`), or a function that returns a the filepath or standalone object:

`configOverride` as a file path string:

```javascript
const browserify = require('browserify')
const fs = require('fs')

const lavamoatOpts = {
    config: './lavamoat/lavamoat-config.json'
    configOverride './lavamoat/lavamoat-config-override.json'
}

const bundler = browserify(['./index.js'], {
  plugin: [
    ['lavamoat-browserify', lavamoatOpts]
  ]
})

bundler.bundle()
  .pipe(fs.createWriteStream('./bundle.js'))
```

For configOverride as an `object` or a `function`, follow the same process as `config` above.

LavaMoat automatically applies an override config if it’s present in `./lavamoat/lavamoat-config-override.json` (relative to the current working directory) but not specified in the bundle command:

```javascript
const lavamoatOpts = {
    config: './lavamoat/lavamoat-config.json'
}
```

### Next Steps

#### Add the generated bundle to your HTML

```html
<script src=bundle.js> </script>
```

And voila! Spin up a server and start using your LavaMoat protected Browserify build.

See [lavamoat-browserify-examples](https://github.com/LavaMoat/lavamoat-browserify-examples/) for more usage examples.

#### Watch for config changes with Watchify

LavaMoat automatically re-builds upon any change to `lavamoat-config-override.json` when using watchify:

```
npm i --dev watchify
```

When creating the Browserify bundle, be sure to follow the instructions as per the official [watchify](https://github.com/browserify/watchify) documentation. Here is an example using LavaMoat:

```javascript
const browserify = require('browserify')
const watchify = require('watchify')
const fs = require('fs')

const lavamoatOpts = {
    writeAutoConfig: true
}

const bundler = browserify({
    entries: ['./index.js'],
    cache: {},
    packageCache: {},
    plugin: [
        ['lavamoat-browserify', lavamoatOpts],
        [watchify]
    ]
})

bundler.on('update', bundle)
bundle()

function bundle() {
    bundler.bundle()
    .pipe(fs.createWriteStream('./bundle.js'))
}
```

**IMPORTANT**: Watchify only watches files that are a part of the build's main require tree found in the entry file, therefore `lavamoat-config-override.json` is not watched by default. The workaround is to emit a `file` event from `bundler` containing the path to the override config. This makes Watchify aware of the file.

```javascript
bundler.emit(file, "./lavamoat/lavamoat-config.json")
```

Now, when changes to `lavamoat-config-override.json` are made, a new bundle is created and the original `lavamoat-config.json` is overwritten. 


#### Config file layout

Here is an example of the config file with each field explained:

```
{
    “resources": {
        “<root>”: {  
            "packages": {  
                “react”: true,   
                “react-dom”: true
            },
        },
        “react”: {
            "globals": {  
                “console”: true 
            }
        }
    }
}
```

* `"resources"`: Packages in your dependency graph. Parses and filled automatically with `writeAutoConfig`.
* `"<root>"`: Entry point for all packages - such as `index.js` - which is considered a package itself. Same as Browserify's entry point.
* `"packages"`: All packages accessible by parent resource. In this example, `"<root>"` has access to packages `"react"` & `"react-dom"`, where `true` means accessible. This means that `"<root>"` can `require` and modify the `exports` of `"react"` & `"react-dom"`.
* `"globals"`: All platform APIs and global variables accessible by parent resource. In this example, `"react"` has access to the global `console`. Any code within `"react"` can therefore call `console.log()`, `console.error()`, etc.


### Introduction to LavaMoat Video

[![introduction to LavaMoat](https://img.youtube.com/vi/pOTEJy_FqIA/0.jpg)](https://www.youtube.com/watch?v=pOTEJy_FqIA)
