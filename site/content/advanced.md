+++
title = "Advanced"
description = "Advanced topics"
weight = 4
+++

## JavaScript interoperability

Trunk will create the necessary JavaScript code to bootstrap and run the WebAssembly based application. It will also
include all JavaScript snippets generated by `wasm-bindgen` for interfacing with JavaScript functionality.

By default, functions exported from Rust, using `wasm-bingen`, can be accessed in the JavaScript code through the global
variable `window.wasmBindings`. This behavior can be disabled, and the name can be customized. For more information
see the [`rust` asset type](@/assets.md#rust).

## Library crate

Aside from having a `main` function, it is also possible to up your project as a `cdylib` project. In order to do that,
add the following to your `Cargo.toml`:

```toml
[lib]
crate-type = ["cdylib", "rlib"]
```

And then, define the entrypoint in your `lib.rs` like (does not need to be `async`):

```rust
#[wasm_bindgen(start)]
pub async fn run() {}
```

## Initializer

Since: `0.19.0-alpha.1`.

Trunk supports tapping into the initialization process of the WASM application. By
default, this is not active and works the same way as with previous versions.

The default process is that trunk injects a small JavaScript snippet, which imports the JavaScript loader generated
by `wasm_bindgen` and calls the `init` method. That will fetch the WASM blob and run it.

The downside with is, that during this process, there's no feedback to the user. Neither when it takes a bit longer to
load the WASM file, nor when something goes wrong.

Now it is possible to tap into this process by setting `data-initializer` to a JavaScript module file. This module file
is required to (default) export a function, which returns the "initializer" instance. Here is an example:

```javascript
export default function myInitializer () {
  return {
    onStart: () => {
      // called when the loading starts
    },
    onProgress: ({current, total}) => {
      // the progress while loading, will be called periodically.
      // "current" will contain the number of bytes of the WASM already loaded
      // "total" will either contain the total number of bytes expected for the WASM, or if the server did not provide
      //   the content-length header it will contain 0.
    },
    onComplete: () => {
      // called when the initialization is complete (successfully or failed)
    },
    onSuccess: (wasm) => {
      // called when the initialization is completed successfully, receives the `wasm` instance
    },
    onFailure: (error) => {
      // called when the initialization is completed with an error, receives the `error`
    }
  }
};
```

For a full example, see: <https://github.com/trunk-rs/trunk/examples/initializer>.

## Update check

Since: `0.19.0-alpha.2`.

Trunk has an update check built in. By default, it will check the `trunk` crate on `crates.io` for a newer
(non pre-release) version. If one is found, the information will be shown in the command line.

This check can be disabled entirely, but not enabled the cargo feature `update_check`. It can also be disabled during
runtime using the environment variable `TRUNK_SKIP_VERSION_CHECK`, or using the command line switch
`--skip-version-check`.

The check is only performed every 24 hours.

## Base URLs, public URLs, paths & reverse proxies

Since: `0.19.0-alpha.3`.

Originally `trunk` had a single `--public-url`, which allowed to set the base URL of the hosted application.
Plain and simple. This was a prefix for all URLs generated and acted as a base for `trunk serve`.

Unfortunately, life isn't that simple and naming is hard.

Today `trunk` was three paths:

* The "public base URL": acting as a prefix for all generated URLs
* The "serve base": acting as a scope/prefix for all things served by `trunk serve`
* The "websocket base": acting as a base path for the auto-reload websocket

All three can be configured, but there are reasonable defaults in place. By default, the serve base and websocket base
default to the absolute path of the public base. The public base will have a slash appended if it doesn't have one. The
public base can be one of:

* Unset/nothing/default (meaning `/`)
* An absolute URL (e.g. `http://domain/path/app`)
* An absolute path (e.g. `/path/app`)
* A relative path (e.g. `foo` or `./`)

If the public base is an absolute URL, then the path of that URL will be used as serve and websocket base. If the public
base is a relative path, then it will be turned into an absolute one. Both approaches might result in a dysfunctional
application, based on your environment. There will be a warning on the console. However, by providing an explicit
value using serve-base or ws-base, this can be fixed.

Why is this necessary and when is it useful? It's mostly there to provide all the knobs/configurations for the case
that weren't considered. The magic of public-url worked for many, but not for all. To support such cases, it
is now possible to tweak all the settings, at the cost of more complexity. Having reasonable defaults should keep it
simple for the simple cases.

An example use case is a reverse proxy *in front* of `trunk serve`, which can't be configured to serve the trunk
websocket at the location `trunk serve` expects it. Now, it is possible to have `--public-url` to choose the base when
generating links, so that it looks correct when being served by the proxy. But also use `--serve-base /` to keep
serving resource from the root.
