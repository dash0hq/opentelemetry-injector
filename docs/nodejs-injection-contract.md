# Node.js Injection Contract — at a glance

**Activation:** the injector prepends `--require <entry point>` to `NODE_OPTIONS`. This addition is the whole interface.

| Injector | Injectable distribution |
| --- | --- |
| Resolves the entry point from its own config | Ships an entry point it owns, at a stable path |
| Checks the entry point exists and is readable | Reaches upstream via `exports` subpaths, never internal build paths |
| Recognises Node.js processes | Checks the Node.js version, stands down if unsupported |
| Adds exactly one `--require`, preserving existing `NODE_OPTIONS` | Loads twice without double-instrumenting |
| Skips if its flag is already there | Stands down if an SDK is already active |
| Skips if disabled in injector config | Reads `OTEL_SDK_DISABLED` / `OTEL_CONFIG_FILE` itself |
| Reads no SDK config to decide activation¹ | Resolves every dependency from inside its own tree |
| Says why on stderr when what was configured cannot be used² | Says why on stderr, whenever it stands down |

¹ It does *contribute* one: host attributes are merged into `OTEL_RESOURCE_ATTRIBUTES`, never overwriting keys that
are already set. Read only to merge, never to decide whether to activate.

² Subject to the injector's log level, whose default is `error` — so the severity is part of the requirement, not an
implementation detail. Skips that mean *nothing was asked* (not a Node.js process, nothing configured, switched off)
stay quiet at debug or info. Skips that mean *what was asked could not be delivered* must be reported; whether that
has to clear the default log level is still open.

## Entry point resolution

1. `<distribution_root>/register.js` exists and is readable → `--require <distribution_root>/register.js`
2. else **skip**, and log why

One fixed filename, one `stat`. The injector never parses `package.json`, never resolves `main`, never reads anything
inside the distribution.

`register.js` must be **CommonJS**. An ESM entry point aborts the application on Node.js 18, and on *every* version if
it uses top-level await (`ERR_REQUIRE_ASYNC_MODULE`).

That is a constraint on *loading*, not on coverage: `module.register()` installs the ESM loader hook at runtime and can
be called from a CommonJS file, so the distribution must instrument **CommonJS and ESM applications alike**. Upstream
does not yet — an ESM app gets core-module spans only, which looks like working instrumentation.

## Neither side may break the app

| | Allowed | Never |
| --- | --- | --- |
| Can't resolve the entry point | Skip injection | Inject anyway |
| Unsupported Node.js version | Run uninstrumented | Throw |
| Missing or invalid SDK config | Run uninstrumented | Throw |
| Broken dependency in the distribution | Run uninstrumented | Throw |
| Shutdown flush fails | Exit normally | Change exit code |
| Collector unreachable at shutdown | Give up on a deadline, exit³ | Block termination indefinitely |

**Everything that fails during `--require` kills the application before its first line.** Degrade, never fail.

³ A few hundred milliseconds is fine, seconds are not: the wait is paid on every rolling update and every scale-down.
Losing the last batch of spans is better than a process that will not stop.

## Changing the contract

| Change | Breaking? |
| --- | --- |
| What runs behind the entry point | No |
| Moving or renaming the entry point | **Yes** — needs a major bump + a forwarding shim |
| Raising the minimum Node.js version | **Yes** |
