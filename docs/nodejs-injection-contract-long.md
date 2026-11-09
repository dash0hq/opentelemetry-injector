# Node.js Injection Mechanism Contract

Status: **draft / proposal**

Normative basis:
[Automatic Injection](https://github.com/open-telemetry/opentelemetry-specification/pull/5308), a pending addition to
the OpenTelemetry [library guidelines](https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/library-guidelines.md)
(status: Development), which
[fixes open-telemetry/opentelemetry-specification#5267](https://github.com/open-telemetry/opentelemetry-specification/issues/5267).

Tracking issue: [open-telemetry/opentelemetry-injector#451](https://github.com/open-telemetry/opentelemetry-injector/issues/451)

The specification requires that an *injectable distribution* document "their activation mechanism and all prerequisites
for safe activation and operation in an injection mechanism contract", and that the contract "identify which
prerequisites require verification by the injector tool and which compatibility checks the distribution performs
itself". This document is the OpenTelemetry Injector's side of that requirement for Node.js: it states the activation
mechanism the injector implements, the prerequisites it is willing to verify, and what it therefore requires a Node.js
injectable distribution's own contract to assume responsibility for.

The key words "MUST", "MUST NOT", "REQUIRED", "SHOULD", "SHOULD NOT", and "MAY" are to be interpreted as described in
[RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

## Motivation

The injector's Node.js support has, until now, had no injection mechanism contract, written or otherwise. In practice it
had an *implied* activation mechanism: the configuration value `nodejs_auto_instrumentation_agent_path` pointed at
`<install dir>/node_modules/@opentelemetry/auto-instrumentations-node/build/src/register.js`, an internal build artifact
path inside a third-party npm package.

`@opentelemetry/auto-instrumentations-node`
[v0.82.0](https://github.com/open-telemetry/opentelemetry-js-contrib/blob/main/packages/auto-instrumentations-node/CHANGELOG.md#0820-2026-10-20)
(2026-10-20) moved that file to `dist/register.cjs`. From the perspective of the upstream package this was not a
breaking change: the supported entry point is the `exports` subpath `@opentelemetry/auto-instrumentations-node/register`,
and the build layout underneath is private. From the perspective of every injector in the field it was a hard break,
because `--require` takes a filesystem path, and a bare package specifier is not something an injector can hand to it.

That break is now contained, by a mechanism the upstream maintainer has declared stable. A *bare specifier* is not
resolvable by `--require`, but it is resolvable *from inside* a CommonJS entry point against an arbitrary installation
directory, and that resolution does honor `exports`:

```js
require.resolve('@opentelemetry/auto-instrumentations-node/register', { paths: ['/path/to/installation'] })
// → '/path/to/installation/node_modules/@opentelemetry/auto-instrumentations-node/dist/register.cjs'
```

The packaged distribution's entry point adopted this in
[opentelemetry-packaging#166](https://github.com/open-telemetry/opentelemetry-packaging/pull/166), and the maintainer
of `@opentelemetry/auto-instrumentations-node`
[confirmed](https://github.com/open-telemetry/opentelemetry-injector/issues/451#issuecomment-6669882115) that it "will
be stable". The published subpaths are therefore the durable interface to the upstream package, and
[Activation mechanism requirements](#activation-mechanism-requirements) makes using them a requirement rather than a
good idea.

What this does *not* fix is the manually configured injector: an operator who points
`nodejs_auto_instrumentation_agent_path` straight at an upstream internal path, or who upgrades the npm package
themselves, still breaks on upgrade.
[opentelemetry-js-contrib#3878](https://github.com/open-telemetry/opentelemetry-js-contrib/pull/3878) proposes
republishing `build/src/register.js` as a forwarding stub for a transition period, which is exactly the remedy
[Versioning and evolution](#versioning-and-evolution) asks of a distribution that moves its entry point.

The same release raised the minimum supported Node.js runtime to 22.15.0, dropping 18 and 20 — a prerequisite tightened
"in ways that make previously supported applications incompatible", which the specification classes as a
backward-incompatible change to the contract. The injector, by design, activates distributions in *every* Node.js
process on the machine, including ones running older runtimes it does not control.

Both are the same class of problem: the activation mechanism was never documented, the prerequisites were never
enumerated, and responsibility for verifying them was never divided. So neither side knew what it was allowed to change.

## Scope

In scope: the activation mechanism for Node.js injectable distributions, the prerequisites for safe activation, and the
division of verification responsibility between the injector and the distribution.

Out of scope: the behavior of the instrumentation itself — which instrumentations are enabled, how the SDK is
configured, which exporters are used. Those are governed by the OpenTelemetry SDK and configuration specifications.
However, the OpenTelemetry SDK and the automatic instrumentations SHOULD endeavour to provide as stable and consistent a
basis as possible in terms of the telemetry generated: supporting stable semantic conventions by default, avoiding
retiring instrumentations outside major releases, and likely more principles yet to be identified.

## Activation mechanism

The injector activates a Node.js injectable distribution by adding a `--require <entry point>` flag to the
`NODE_OPTIONS` environment variable of the host process.

The injector sets `--require` rather than `--import` because it is the only form that serves both CommonJS and ES module
applications: a CommonJS entry point loaded via `--require` can register ESM loader hooks itself via `module.register()`,
whereas an `--import`ed entry point imposes an ESM-only load path. The consequence for distributions is in
[Activation mechanism requirements](#activation-mechanism-requirements).

## Prerequisites and division of verification

The specification requires this contract to identify which prerequisites the injector tool verifies and which
compatibility checks the distribution performs itself. For Node.js:

| Prerequisite | Verified by | Reference |
| --- | --- | --- |
| The entry point exists and is accessible | **Injector** | [Prerequisite verification](#prerequisite-verification) |
| Node.js instrumentation has not been disabled by injector-based configuration | **Injector** | [Prerequisite verification](#prerequisite-verification) |
| The distribution is not already activated in this process | **Injector** (flag deduplication) and **distribution** (idempotent load) | [Idempotent activation](#idempotent-activation), [Idempotent loading](#idempotent-loading) |
| The host runtime is Node.js | **Injector** (executable matching) | [Prerequisite verification](#prerequisite-verification) |
| The host runtime is a Node.js version the distribution supports | **Distribution** (fail safe) | [Compatibility checks](#compatibility-checks) |
| The distribution's own dependencies are resolvable and loadable | **Distribution** | [Self-containment](#self-containment) |
| No OpenTelemetry SDK is already active in the host process | **Distribution** | [Idempotent loading](#idempotent-loading) |
| The OpenTelemetry SDK is enabled and its configuration is usable | **Distribution** | [Declarative configuration](#declarative-configuration) |
| CPU architecture and C library flavor compatibility | **Not applicable** | [Build selection](#build-selection) |

The injector verifies only prerequisites it can evaluate from the filesystem, the host runtime's own binary (including
but not limited to ELF symbols), the `/proc` filesystem, and the injector's own configuration. It does not execute,
read, or parse the entry point or any Node.js code.

Specifically:
- Establishing *that* a host is a Node.js process is the injector's responsibility, from the executable path and
  command line it already reads.
- Establishing *which version* of Node.js it is belongs to the distribution, which has `process.versions` in hand the
  moment it loads.

Why the version check sits with the distribution for Node.js, while its Java counterpart sits with the injector, is in
[Appendix A](#appendix-a-why-the-nodejs-version-check-sits-with-the-distribution).

Regarding configuration, the contract distinguishes two layers:

- **Injector configuration** — `injector.conf`, its `conf.d` drop-ins, and the corresponding environment variables —
  governs *activation*: whether a distribution is activated into this process at all, and which one.
- **OpenTelemetry SDK configuration** — environment variables such as `OTEL_SDK_DISABLED`, and declarative
  configuration via `OTEL_CONFIG_FILE` — governs the *behavior of the SDK after activation*. The injector MUST NOT
  treat any of it as a prerequisite for activation.

One carve-out: the injector *contributes* SDK configuration even though it never consults it to decide activation. It
merges resource attributes it can determine about the host into `OTEL_RESOURCE_ATTRIBUTES`
(`src/resource_attributes.zig`, `src/root.zig`), in a strict precedence order:

1. key-value pairs already present in `OTEL_RESOURCE_ATTRIBUTES` — highest priority, never overwritten;
2. then `OTEL_INJECTOR_RESOURCE_ATTRIBUTES`, the injector's own operator-facing input;
3. then the injector's built-in host mappings.

A key written by an earlier step is skipped by every later one, so whatever the application or operator set explicitly
survives verbatim. The injector's contribution is a set of *defaults*, not an override.

This is a write, in the same spirit as the `NODE_OPTIONS` write. The variable is read only to merge into it, never to
decide *whether* to activate. The distribution MUST treat the resulting `OTEL_RESOURCE_ATTRIBUTES` as ordinary SDK
configuration and MUST NOT assume it is absent, nor that it reflects only what the application set.

The boundary is a deliberate separation of concerns. `OTEL_SDK_DISABLED=true`, for instance, is an environment variable
the injector already has in hand and could trivially short-circuit on, yet it is not an activation prerequisite: the SDK's
disabled state is defined as a no-op SDK, not an absent one, and an injector that second-guesses SDK semantics acquires a
second, divergent implementation of them. The distribution activates, reads its own configuration, and disables its own
instrumentation.

### Build selection

The specification requires that a contract document "deterministic logic for selecting a compatible build" where
multiple builds are published. A Node.js injectable distribution is expected to be a single, architecture- and
libc-independent build, so no selection logic applies, and the injector implements none for Node.js.

This holds for the packaged distribution as built today. Installing
`@opentelemetry/auto-instrumentations-node` the way the packaging project does — `npm install --ignore-scripts` of the
packed tarball — yields a tree containing no compiled addons (`*.node`), no `prebuilds` directories, and no package
declaring a `cpu`, `libc`, or `gypfile` constraint; the resulting trees for `linux/x64` and `linux/arm64` are
byte-identical. This follows from what the package does: instrumentations patch libraries at require time and never
bundle them, so native addons in an instrumented process belong to the *application's* dependency tree, built for the
application's architecture, and are never artifacts of the distribution.

A distribution that ships native addons, and therefore needs per-architecture or per-libc builds, requires a revision
of this contract and a corresponding bump of the packaging interface generation — the mechanism described under
[Injector interface versioning](https://github.com/open-telemetry/opentelemetry-packaging/blob/main/docs/design/packages-meta-architecture.md#injector-interface-versioning)
in the OpenTelemetry Packaging meta-architecture. Each language package carries its interface generation in a virtual
`Provides`, following the Debian shared-library naming convention: the Node.js package today provides
`opentelemetry-nodejs-autoinstrumentation1`, covering "the conf.d key names, the file layout under
`/usr/lib/opentelemetry/<language>/`, and the `otel-config.yaml` structure". Per-architecture builds would break two
properties of that generation at once — the package is `Architecture: all` / `noarch`, and the layout under
`/usr/lib/opentelemetry/nodejs/` is fixed — so such a distribution would provide
`opentelemetry-nodejs-autoinstrumentation2` instead, and the package manager would hold back the upgrade until every
consumer moved with it.

## Requirements on the injector

### Preservation of existing `NODE_OPTIONS`

If `NODE_OPTIONS` is already set, the injector MUST prepend its `--require` flag and preserve the original value
verbatim. It MUST NOT reorder, deduplicate, or otherwise rewrite flags it did not add.

### Prerequisite verification

Before adding the flag, the injector MUST verify that the entry point is accessible, and MUST decline to activate the
distribution if it is not. A `--require` of a nonexistent path aborts the Node.js process at startup.

The injector MUST NOT read, parse, or validate the *contents* of the entry point, and MUST NOT read any other file
inside the distribution — including its `package.json`. Accessibility is the only check. Anything beyond that would
couple the injector to the distribution's implementation and recreate the problem this contract exists to solve.

If the configured path is empty, or Node.js instrumentation is disabled by *injector* configuration — `nodejs` listed
in the `auto_instrumentation_disabled` configuration key or the `OTEL_INJECTOR_AUTO_INSTRUMENTATION_DISABLED`
environment variable, or either set to `*` — the injector MUST NOT modify `NODE_OPTIONS`.

Where the injector establishes that a host is a Node.js process, it does so from the executable path and command-line
arguments it already reads from `/proc/self`, via the same include and exclude pattern evaluation it applies to every
process. (As of the time of writing, it does not: the current implementation has no Node.js-specific recognition, and
sets `NODE_OPTIONS` for every process that passes the operator-configured allow/deny evaluation, relying on the fact
that only Node.js reads the variable; but the Injector SIG has been introducing opt-in detection logic for other
runtimes, so if there is a use-case and a reliable detection method, it can be made to happen.)

That "set it and see what sticks" model is being retired runtime by runtime where it proves unsafe.
[#442](https://github.com/open-telemetry/opentelemetry-injector/pull/442) is the first case: it stops setting
`JAVA_TOOL_OPTIONS` in processes with no JVM launcher loaded, because a wrapper shell that inherits the variable
passes it to an unsupported Java child that then crashes. Node.js has no equivalent hazard today — nothing downstream
of an inherited `NODE_OPTIONS` fails in a way the distribution cannot absorb — and no equivalent marker either: the
`node` executable loads no `libnode.so` (only `libstdc++`, `libm`, `libc` and friends), so the loaded-library test that
works for the JVM has no Node.js counterpart. Recognition would have to come from `/proc/self/exe` directly. Narrowing
activation to actual Node.js processes remains the unachieved stretch goal recorded in [DESIGN.md](../DESIGN.md), not a
requirement this contract introduces.

### Idempotent activation

If the exact `--require <entry point>` flag the injector would add is already present in `NODE_OPTIONS`, the injector
MUST leave the variable unmodified. This prevents double activation when a process inherits an already-injected
environment, for example when a shell entry point is injected and then execs the Node.js binary.

### Diagnostics

When the injector declines to activate a distribution, it MUST emit a diagnostic naming the path it tried and the
reason, satisfying the specification's requirement that a declining injector tool "SHOULD emit actionable diagnostics
explaining why". Silence on a declined activation is indistinguishable from a distribution that failed later, and makes
field support impossible.

A reason is only reported if its severity survives the injector's log level, and the injector's default log level is
`error`: `log_level: LogLevel = .Error` in `src/proc_self_environ_values.zig`, with `printInfo` and `printWarn` in
`src/print.zig` gated on it. A requirement to say why, with the severity left unstated, is therefore satisfiable by an
implementation that says nothing in every default installation. Two cases have to be told apart:

- **Nothing was asked of the injector.** The process is not a Node.js process, no distribution is configured, or
  activation is switched off in injector configuration. Declining is the expected outcome and carries no operator-facing
  information, so the diagnostic belongs at debug or info severity, where the default log level suppresses it. A
  host-wide preload that announced every process it correctly ignored would be unusable.
- **Something was asked and could not be delivered.** A distribution is configured, but its entry point is missing or
  unreadable. The operator's intent is on record and was defeated, and the application will run uninstrumented
  indefinitely with no other signal — the silent failure this contract exists to prevent.

For the second case the injector MUST emit the diagnostic, and MUST document the severity it uses. Whether that severity
has to clear the default log level is [open question 5](#open-questions): making it visible by default is what gives
"say why" its force, but it also conflicts with the severity already chosen for the equivalent JVM case.

Diagnostics MUST go to stderr, on both sides of this contract. The host application's stdout is its own: it may be a
pipeline stage, a JSON log stream, or a protocol channel, and telemetry tooling that writes to it corrupts data the
application's consumers depend on. The injector satisfies this today: `_printMessage` in `src/print.zig` writes through
`std.debug.print`, which goes to stderr.

## Requirements on the injectable distribution

These are the terms a Node.js injectable distribution must meet to be activatable by this injector, and which its own
injection mechanism contract is expected to restate and version.

### Activation mechanism requirements

A distribution MUST expose an entry point at a path that is stable across its releases, and MUST treat that path as part
of its injection mechanism contract: moving or renaming it is a backward-incompatible change, governed by
[Versioning and evolution](#versioning-and-evolution).

The entry point MUST be a file owned by the distribution. A distribution MUST NOT designate a file inside a third-party
dependency — including `@opentelemetry/auto-instrumentations-node` — as its entry point. A dependency's internal build
layout is that dependency's business, is not subject to this contract, and cannot be made subject to it; depending on it
is exactly the failure mode of [#451](https://github.com/open-telemetry/opentelemetry-injector/issues/451).

A distribution that wraps `@opentelemetry/auto-instrumentations-node` MUST therefore ship a thin entry point of its own
that loads the upstream register hook. The packaged distribution does this at
`/usr/lib/opentelemetry/nodejs/register.js`.

That entry point MUST reach the upstream package through the subpaths upstream publishes in its `exports` map,
resolving them against the distribution's own `node_modules` tree:

```js
const resolve = (request) => require.resolve(request, { paths: [modules] });
require(resolve('@opentelemetry/auto-instrumentations-node/register'));
```

It MUST NOT join a path into the dependency's internal build layout. The published subpaths (`.` and `./register`) are
upstream's API and have been
[declared stable](https://github.com/open-telemetry/opentelemetry-injector/issues/451#issuecomment-6669882115) by its
maintainer; the layout behind them — `build/src/register.js` before 0.82.0, `dist/register.cjs` after — is not, and
moved without a major version. A distribution resolving through `exports` absorbs that move silently. One joining paths
must chase every upstream release, and ships a broken entry point whenever it fails to. The packaged distribution
switched to `require.resolve` in
[opentelemetry-packaging#166](https://github.com/open-telemetry/opentelemetry-packaging/pull/166) for exactly this
reason.

The entry point MUST be loadable by `node --require`: CommonJS, with a file extension Node.js resolves as CommonJS
(`.js` under a distribution whose nearest `package.json` is not `"type": "module"`, or `.cjs`). A distribution that
instruments ES module applications MUST do so from within that CommonJS entry point, by registering loader hooks via
`module.register()`.

### Self-containment

The entry point MUST resolve all of its dependencies from inside the distribution root, by explicit path, and MUST NOT
rely on the host process's `NODE_PATH`, working directory, or `node_modules` resolution. This implements the
specification's guidance to "avoid dependency conflicts with the application by minimizing dependencies and isolating or
vendoring dependencies": the host application's dependency tree is not the distribution's, and may contain conflicting
versions of the OpenTelemetry packages.

### Compatibility checks

Per the specification, "when a distribution performs compatibility checks, it MUST NOT prevent application startup or
cause application failure if those checks fail. It MAY disable some or all instrumentation."

The checks assigned to the distribution in
[Prerequisites and division of verification](#prerequisites-and-division-of-verification) are:

- The entry point MUST verify that the host runtime satisfies the minimum Node.js version its contract documents, and
  MUST disable instrumentation quietly if it does not. The injector activates distributions in Node.js processes of all
  versions present on the machine, including versions the distribution never supported, and including versions a
  distribution upgrade dropped. This is not hypothetical: `@opentelemetry/auto-instrumentations-node` v0.82.0 dropped
  Node.js 18 and 20.
- That version check SHOULD precede loading any instrumentation code, so that a syntax or engine-level incompatibility
  in a dependency cannot be reached on an unsupported runtime.
- The entry point MUST catch every error it can raise during load, including failures to resolve its own dependencies,
  and MUST NOT let an exception escape the `--require` phase.
- The entry point MUST evaluate SDK-level configuration itself, and MUST treat an SDK that is disabled, or a
  configuration it cannot use, as grounds for disabling instrumentation rather than for failing. Because the injector
  does not read SDK configuration, a distribution is activated into processes whose SDK configuration is absent,
  disabled, or invalid, and absorbing that is the distribution's responsibility alone.

When a check fails, the distribution SHOULD emit an actionable diagnostic on stderr, at warning severity, prefixed so
the line is attributable.

### Operating without causing application failure

The specification requires that, when the documented prerequisites are satisfied, an injectable distribution "MUST NOT
prevent application startup or cause the application to fail at runtime", noting that "disabling instrumentation does
not constitute application failure". For Node.js, meeting that bar means the entry point:

- MUST NOT alter the host's exit code, including on flush failure.
- MUST NOT leave a persistent `SIGTERM` (or other termination signal) listener installed in a way that suppresses
  Node.js's default terminate behavior. Registering a listener replaces that default, so a distribution that registers
  one and neither exits nor re-raises the signal holds the process *indefinitely*: it stops responding to `docker stop`
  or `systemctl stop` altogether, and dies only when its orchestrator's grace period expires and it is `SIGKILL`ed. From
  the operator's point of view that is an application that failed at runtime.
- MUST NOT write to stdout, at any log level. Diagnostics go to stderr.
- MUST bound any shutdown flush it performs. Termination MUST NOT wait on network I/O without a deadline: when the
  deadline expires the entry point proceeds with termination and abandons the unflushed telemetry, because losing the
  last batch of spans is a smaller failure than a process that will not stop.

The deadline is a trade of telemetry completeness against termination latency, and this contract deliberately does not
fix a number. The guidance is that a few hundred milliseconds is unobjectionable and that seconds are not: a delay on
every shutdown is paid on every rolling update and every scale-down, where it slows pod restarts for an entire fleet.
[opentelemetry-js-contrib#3349](https://github.com/open-telemetry/opentelemetry-js-contrib/issues/3349) is the concrete
case — a 5-second block on an unavailable collector — and sits at the wrong end of that range.

#### Known gaps in upstream `@opentelemetry/auto-instrumentations-node/register`

A distribution cannot meet the requirements above by delegating to the upstream register hook unchanged. As of 0.82.0,
that hook:

- **Holds `SIGTERM`.** Its shutdown handler prevents the process from exiting on the signal
  ([opentelemetry-js-contrib#3808](https://github.com/open-telemetry/opentelemetry-js-contrib/issues/3808)), which is
  the failure mode the second bullet above exists to prevent.
- **Writes diagnostics to stdout.** At the default log level it prints `OpenTelemetry automatic instrumentation
  started successfully` and related lines to stdout, not stderr; at `OTEL_LOG_LEVEL=debug` it emits hundreds of lines
  there. Measured on 0.82.0: 3 stdout lines at info, 222 at debug, 1 on stderr.
- **Can throw during `--require`.** Its own imports are not wrapped, so a resolution or initialization failure
  propagates out of the `--require` phase and aborts the host application.
- **Does not bound its shutdown flush.** With an unreachable collector, the shutdown it performs on termination blocks
  for as long as the exporter keeps retrying — measured at 5 seconds in
  [#3349](https://github.com/open-telemetry/opentelemetry-js-contrib/issues/3349). This is independent of the `SIGTERM`
  gap above: fixing the signal handling makes the process exit, but it still exits late.

A distribution wrapping it MUST therefore compensate in its own `register.js` — guarding the require, managing signal
handlers itself, bounding the shutdown, and keeping upstream's output off stdout — until these are fixed upstream. The
packaged distribution's entry point already guards the require and manages signal handlers for its
declarative-configuration path.

[opentelemetry-js-contrib#3888](https://github.com/open-telemetry/opentelemetry-js-contrib/pull/3888) is open with
fixes for the first three. It does not address the fourth, and the `SIGTERM` fix it proposes awaits the SDK shutdown
before re-raising the signal, so on an unreachable collector it converts a process that never stops into one that stops
late — the gap moves rather than closing.

### Idempotent loading

Loading the entry point more than once in a single process MUST be safe, and MUST NOT result in duplicate
instrumentation. The injector deduplicates its own flag ([Idempotent activation](#idempotent-activation)) but cannot prevent an operator or base
image from adding a second `--require` by hand.

A distribution SHOULD also detect that an OpenTelemetry SDK is already active in the host process and disable its own
instrumentation, implementing the specification's guidance to "design injectable distributions and their SDKs to coexist
with existing SDKs and instrumentation".

### Declarative configuration

A distribution SHOULD support
[declarative configuration](https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/configuration/README.md#declarative-configuration),
per the specification, which further notes that declarative configuration support is a prerequisite for inclusion in the
OpenTelemetry System Packages project.

How a distribution routes between declarative and environment-variable configuration is internal to it and not
constrained by this contract. The packaged distribution's entry point, for example, boots through `startNodeSDK` when
`OTEL_CONFIG_FILE` is set and defers to the upstream register hook otherwise; the injector neither knows nor cares.

What the contract does constrain is that declarative configuration MUST NOT become a precondition for activation. A
distribution MUST remain activatable, and MUST instrument the host application, with no `OTEL_CONFIG_FILE` set. Where
one is set but the file is missing, unreadable, or invalid, the distribution MUST fall back to leaving the application
uninstrumented, never to failing it — the `--require` phase runs before the application's first line, so an exception
raised while parsing a configuration file would abort a process that would otherwise have started. The packaged
distribution's entry point already treats its entire declarative-configuration path this way.

## Resolution and configuration

### Current state

The injector exposes the entry point as a file path:

- configuration key `nodejs_auto_instrumentation_agent_path`
- environment variable `NODEJS_AUTO_INSTRUMENTATION_AGENT_PATH`

with an empty default; a distribution supplies the value via a drop-in configuration file. The packaged distribution
sets it to `/usr/lib/opentelemetry/nodejs/register.js`.

This key is misdescribed in the injector's own documentation as "the path to an installation of the npm module
`@opentelemetry/auto-instrumentations-node`", which is a directory, while the value is in fact a file inside such an
installation. That conflation is how an upstream-internal path ended up baked into the injector's documentation, sample
configuration, and tests.

### Proposed: resolve a distribution root, not a file

The injector SHOULD move to configuring the *distribution root* and deriving the entry point from it, matching how every
other runtime in the injector is configured (`python_auto_instrumentation_agent_path_prefix`,
`ruby_auto_instrumentation_agent_path_prefix`, `dotnet_auto_instrumentation_agent_path_prefix` — each a directory with a
required layout and a well-known entry point within it). Node.js is the sole runtime configured as a bare file path, and
the sole runtime that broke when a third party moved a file.

This is now hardening rather than firefighting. With the packaged distribution resolving upstream through `exports`
and the upstream subpaths declared stable ([Motivation](#motivation)), the incident behind
[#451](https://github.com/open-telemetry/opentelemetry-injector/issues/451) is contained without any change to how the
injector resolves the entry point. What remains is that the injector↔distribution boundary is still a single
hardcoded file path in a configuration file, which is a weaker interface than the one the distribution now enjoys
toward its own dependency. The proposal below closes that gap; it is not required to keep today's packages working.

The entry point is resolved by **one fixed filename**: given root `R`, the entry point is `R/register.js`. This
requires nothing of the distribution beyond a file at a known name, and is satisfied by the packaged distribution
today.

#### Resolution algorithm

The steps below are normative, and the behavior they rely on is measured in
[Appendix B](#appendix-b-resolution-prototype-results). For a configured distribution root `R`, the injector MUST:

1. If `R/register.js` resolves to an existing, readable regular file, activate with `--require R/register.js`.
2. Otherwise, decline, and emit a diagnostic naming `R` ([Diagnostics](#diagnostics)).

That is the entire algorithm. The injector MUST NOT parse `R/package.json`, MUST NOT resolve a `main` field, and MUST
NOT reproduce any part of Node.js's `LOAD_AS_DIRECTORY` resolution. One `stat` on one path is the whole of it, which
keeps [Prerequisite verification](#prerequisite-verification) absolute: accessibility is the only check, and the
injector reads nothing inside the distribution.

#### Why not directory require

An earlier draft of this contract added a second step: `--require R`, where `R/package.json` declares a `main` naming
the entry point. It is the shape [#451](https://github.com/open-telemetry/opentelemetry-injector/issues/451) gestures
at when it suggests "a different package where the *top-level* export is 'register' functionality". It is **rejected**,
for three reasons.

**It cannot distinguish a hook from a library.** This is the decisive one. Point the rule at an
`@opentelemetry/auto-instrumentations-node` installation and it resolves `main` to `./dist/index.cjs` — a module that
exports `getNodeAutoInstrumentations` and registers nothing. Requiring it satisfies every check the injector could
make, costs the host a module load, and leaves the application **uninstrumented with no error anywhere**. Measured:
requiring `dist/index.cjs` leaves the OpenTelemetry API global absent, while requiring `dist/register.cjs` installs
it. A misconfiguration that fails loudly is recoverable; one that silently produces no telemetry is not.

**It costs a `package.json` parse on every exec.** The injector runs in the startup path of every process on the
machine. A JSON read-and-parse to discover something a fixed filename already settles is work charged to every `exec`
on the host, for no capability the fixed filename lacks.

**Its failure modes are sharp.** Node.js resolves `main` far more liberally than an injector should — extensionless
values, values naming a directory, and values escaping the distribution root all resolve — while an `exports`-only
manifest, a `main` naming a missing file, and a malformed `package.json` each abort the host application before its
first line. Reproducing the permissive half faithfully means a second, drifting copy of Node.js's resolver in a hot
path; not reproducing it means the injector is stricter than Node.js in ways distributions must learn. Either way the
rule is a liability the fixed filename does not carry.

The measurements behind all three are in [Appendix B](#appendix-b-resolution-prototype-results).

#### What this requires of a distribution

A distribution MUST place a file named `register.js` at the root of its distribution directory, and that file MUST be
the activation hook — the thing that installs instrumentation, not a library that exports the means to.

`register.js` MUST be CommonJS, as already required by
[Activation mechanism requirements](#activation-mechanism-requirements). This is where that requirement bites hardest,
in two version-dependent ways:

- An ESM entry point aborts the application on Node.js 18, while loading successfully on 20.19+ and 22+, which support
  `require(esm)`. A distribution validated only on a current runtime ships a defect visible exclusively on older
  hosts — precisely the hosts an injector is most likely to find.
- An ESM entry point using **top-level await** aborts the application on *every* version, with
  `ERR_REQUIRE_ASYNC_MODULE`, including those that otherwise support `require(esm)`.

### Backwards compatibility of the configuration key

`nodejs_auto_instrumentation_agent_path` and `NODEJS_AUTO_INSTRUMENTATION_AGENT_PATH` MUST continue to accept a file
path for as long as the injector supports the version range in which they were the only option. Deployed injectors,
operator-rendered configurations, and vendor distributions carry these values today; silently reinterpreting an existing
file path as a directory would break them.

If a new root-oriented key is introduced, the injector MUST prefer an explicitly set legacy key over a default-valued
new one, and MUST emit a deprecation diagnostic when the legacy key is used.

## Versioning and evolution

The specification requires that an injection mechanism contract "MUST be subject to semantic versioning", and that for
stable distributions "backward-incompatible changes to the contract MUST require a major version increment", explicitly
including "tightening prerequisites in ways that make previously supported applications incompatible".

Applied here:

- A distribution MAY change anything behind its entry point — which upstream version it vendors, how it loads it, what
  it configures — without consulting the injector or incrementing anything.
- Moving or renaming the entry point is backward-incompatible.
- Raising the minimum supported Node.js version is backward-incompatible, by the specification's own example.
- The injector MAY change *how* it computes the entry point path from configuration, provided every previously valid
  configuration continues to resolve to the same entry point.
- Neither side may rely on the other's internals. The entry point — its path, and the `main` entry of the distribution
  root's `package.json` where one is used to resolve it — is the entirety of the interface.

Beyond the version increment, a distribution making a backward-incompatible change to its activation mechanism SHOULD
keep the previous entry point working as a forwarding shim for at least one minor release line. The reason is specific
to injector tools and is the lesson of [#451](https://github.com/open-telemetry/opentelemetry-injector/issues/451):
injector binaries and their configuration live on long-lived hosts and in long-lived container images, so correcting a
path in the distribution's source repository does not correct the injectors already deployed against the old one. Only a
forwarding shim does.

[opentelemetry-js-contrib#3878](https://github.com/open-telemetry/opentelemetry-js-contrib/pull/3878) is that remedy
applied to the case that prompted this contract: it republishes `build/src/register.js` as a forwarding stub, so
installations and injector configurations pinned to the pre-0.82.0 path keep working through a transition period. It is
worth noting what it does *not* undo — the move itself stands, and a distribution that resolves through `exports`
([Activation mechanism requirements](#activation-mechanism-requirements)) never needed the stub. The stub exists for
the configurations that bypassed the interface, which is the population a forwarding shim is always for.

That population cannot shrink on its own yet. The injector's configuration key still takes a path to a *file*, not a
distribution root ([Current state](#current-state)), so an operator pointing it at an upstream internal path has no
supported alternative to migrate to. The stub therefore needs to outlive the usual one-minor-line guidance: it should
stay until the distribution-root migration lands and operators have a root to point at instead.

This document is versioned with the injector. A distribution declares the contract version it satisfies in its own
documentation. The injector SHOULD NOT attempt to detect the contract version at runtime: runtime negotiation would
require reading the distribution's contents, which [Prerequisite verification](#prerequisite-verification) forbids.

## Conformance

An injectable distribution conforms if, on a runtime satisfying its documented prerequisites:

1. `node --require <entry point> app.js` starts `app.js` and instruments it.
2. On a runtime *outside* its documented prerequisites, the same command starts `app.js`, leaves it uninstrumented,
   exits with the same code as the uninstrumented run, and emits one attributable warning on stderr.
3. `node --require <entry point> --require <entry point> app.js` behaves identically to a single `--require`.
4. The entry point loads with the distribution root read-only, from an arbitrary working directory, with `NODE_PATH`
   unset, and with a host `node_modules` containing conflicting OpenTelemetry package versions.
5. A `SIGTERM` sent to the instrumented process terminates it in the same manner as the uninstrumented process, and
   within a bounded delay of it. The delay is bounded with the collector *unreachable*, not merely absent: a test that
   exports nowhere never exercises the flush path that produces the delay.
6. Deleting or corrupting a dependency inside the distribution root causes the application to start uninstrumented, not
   to fail.
7. The application starts and is instrumented with no `OTEL_CONFIG_FILE` set, and starts uninstrumented — with the same
   exit code as the uninstrumented run — when `OTEL_CONFIG_FILE` points at a missing or invalid file.
8. With `OTEL_SDK_DISABLED=true`, the application starts, emits no telemetry, and exits with the same code as the
   uninstrumented run.
9. `register.js` is CommonJS: it loads on the distribution's minimum supported Node.js version and on every later one,
   including versions without `require(esm)`.
10. The instrumented application's stdout is byte-identical to the uninstrumented run's, at every `OTEL_LOG_LEVEL`.

The injector conforms if, for every supported configuration form:

1. It adds exactly one `--require` for the resolved entry point.
2. It preserves pre-existing `NODE_OPTIONS` content and ordering.
3. It is a no-op when its own flag is already present.
4. It declines to activate, with a diagnostic at its documented severity, when the entry point is inaccessible.
5. It never reads any file inside the distribution — not the entry point, not `package.json`.
6. It activates identically regardless of SDK-level configuration: `OTEL_SDK_DISABLED`, `OTEL_CONFIG_FILE` (set, unset,
   or pointing at an invalid file), and `OTEL_*` variables generally produce the same `NODE_OPTIONS` result.
7. Given a distribution root with no `register.js`, it declines with a diagnostic, whatever else the root contains.
8. Every diagnostic it emits goes to stderr.
9. Declining because nothing was configured is silent at the default log level; declining because what was configured
   could not be used is reported at the severity the injector documents, and that severity is asserted by the test, not
   left to whatever the implementation happens to pass.

Injector-side conformance is testable in `injector-integration-tests/tests/nodejs.tests`. Those tests currently assert
the upstream-internal `…/@opentelemetry/auto-instrumentations-node/build/src/register.js` path as a literal, as do
`injector-integration-tests/common/conf.d/nodejs.conf`, the unit tests in `src/nodejs.zig` and `src/config.zig`, and the
examples in `README.md`. All of these encode the pre-contract assumption and need to be updated to a
distribution-owned path as part of adopting this document.

## Open questions

1. **New configuration key, or redefine the existing one?** Introducing
   `nodejs_auto_instrumentation_agent_path_prefix` aligns Node.js with the other runtimes but leaves two keys to support
   indefinitely. Teaching the existing key to accept either a file or a directory is less to document but muddies a key
   whose name already caused confusion.
2. **Should the injector verify the Node.js version itself, as it is coming to do for Java?** The specification permits
   enabling by default only where the injector tool can verify the prerequisites assigned to it. This contract assigns
   the version prerequisite to the distribution, on the failure-mode argument in
   [Prerequisites and division of verification](#prerequisites-and-division-of-verification) — but a distribution that
   ignores [Compatibility checks](#compatibility-checks) will break applications on unsupported runtimes with the
   injector having activated it, and the injector will have had no say.

   [#442](https://github.com/open-telemetry/opentelemetry-injector/pull/442) establishes both the precedent and the
   shape: a `jvm_auto_instrumentation_minimum_java_major_version` configuration key (environment override, default 8,
   set to `0` to restore unconditional injection), a stand-down when the version cannot be determined, and a documented
   list of cases the detection cannot see. A symmetrical
   `nodejs_auto_instrumentation_minimum_node_major_version` is mechanically straightforward — more so than the Java
   case, which must first discover `libjvm.so` by way of `libjli.so`. The `node` binary carries its own version string
   in `.rodata` at a single occurrence (`v22.23.4` in `node:22`, alongside the `node.js/v22.23.4` User-Agent string),
   reachable from `/proc/self/exe` with no library discovery at all. The costs are a `.rodata` scan of a large
   executable (117 MB for Node 22) at every process startup, a second place where a minimum version is configured and
   can drift from the distribution's own, and a new failure surface in the hot path.

   The question is whether defense in depth is worth that, given that — unlike the JVM — a Node.js distribution that
   does its job needs no help from the injector.
3. **Should this contract be upstreamed?** It is written as the injector's contract, but nearly all of
   [Requirements on the injectable distribution](#requirements-on-the-injectable-distribution) is language-specific elaboration of
   [#5308](https://github.com/open-telemetry/opentelemetry-specification/pull/5308) that any injector tool activating a
   Node.js distribution would need — the Operator included. It may belong closer to the specification, or in
   opentelemetry-js-contrib alongside the distribution it mostly constrains.
4. **Does this generalize across the injector's other runtimes?** The same reasoning applies to the JVM, Python, Ruby,
   and .NET paths, and [Build selection](#build-selection) is the only section where Node.js differs materially from runtimes
   that do ship per-architecture and per-libc builds. If the shape holds, this document becomes the first of a family,
   or a per-runtime section of a single contract document.
5. **At what severity does a defeated activation get reported?** [Diagnostics](#diagnostics) requires the injector to
   say why it declined when a distribution was configured but could not be used, and to document the severity, but stops
   short of requiring a severity the default log level lets through. The case for requiring it: the injector's default
   log level is `error`, so an info or warn diagnostic is invisible in a default installation, and the operator's only
   remaining evidence that their configuration was defeated is the absence of telemetry — which looks identical to an
   application that was never instrumented, a collector that is down, or a sampling decision. The case against: the
   severity for the equivalent JVM situation, skipping on an unsupported JVM version, was settled at info or warn, and
   an injector active on every process of a host would emit error-level output for every short-lived process on a
   machine that simply has an old runtime — noise that trains operators to ignore the channel.

   The distinction that may resolve it is whether the distribution was named explicitly in injector configuration or
   merely discovered: an explicitly configured path that does not resolve is a mistake someone made and wants to hear
   about, while an unsupported runtime is a property of the host. Settling this needs the Java side in the room, since
   diverging per runtime is worse than either answer.

## Appendix A: Why the Node.js version check sits with the distribution

The runtime-version prerequisite is assigned to the distribution for Node.js
([Prerequisites and division of verification](#prerequisites-and-division-of-verification)) and to the injector for
Java ([#442](https://github.com/open-telemetry/opentelemetry-injector/pull/442)). The asymmetry is a difference in
failure mode, not a difference in capability.

A JVM "will fail to initialize an agent that's compiled with a newer class version", as #442 puts it. An incompatible
Java agent therefore kills the application with `UnsupportedClassVersionError` before any guard inside the agent could
run, and the check has nowhere to live but the injector. A Node.js entry point, by contrast, is ordinary JavaScript
evaluated by an interpreter that has already started successfully: it can read `process.versions`, decline, and leave
the application running. The same PR names Node.js among the runtimes where "we [can] build a … version guard in the
packaging scripts".

Capability is not the constraint. The injector already inspects host runtime binaries, and doing so for Node.js would
be *easier* than for Java: #442 must first locate `libjvm.so` by way of the loaded `libjli.so`, whereas the `node`
binary carries its own version string in `.rodata` at a single occurrence (`v22.23.4` in `node:22`, alongside the
`node.js/v22.23.4` User-Agent string), reachable from `/proc/self/exe` with no library discovery at all. What Node.js
lacks is a marker library: `node` loads no `libnode.so`, only `libstdc++`, `libm`, `libc` and friends, so the
loaded-library test #442 uses to recognise JVM processes has no Node.js counterpart.

Whether the injector should nonetheless verify the Node.js version as defense in depth is
[Open questions](#open-questions), question 2.

## Appendix B: Resolution prototype results

The [resolution algorithm](#resolution-algorithm) was prototyped before being written down, against Node.js 18.20.8,
20.20.2, 22.23.4 and 24.22.2 (linux/arm64), and against a real packaged distribution root built the way the packaging
project builds it (`npm pack` + `npm install --ignore-scripts` of
`@opentelemetry/auto-instrumentations-node@0.82.0`).

### Directory-require behavior

Each row is a distribution root shape, invoked as `node --require <root> app.js`. "Loaded" names the file Node.js
actually executed; "app ran" is whether the host application reached its first line.

| Root shape | Result | Loaded | App ran |
| --- | --- | --- | --- |
| `register.js` only, no `package.json` | **fails** | — | no |
| `package.json` `main` → existing file | ok | the `main` target | yes |
| `package.json` `main` → missing file | **fails** | — | no |
| `package.json` with `exports` only | **fails** | — | no |
| `package.json` with both `main` and `exports` | ok | the **`main`** target | yes |
| `package.json` without `main`, `index.js` present | ok | `index.js` | yes |
| `package.json` without `main`, no `index.js` | **fails** | — | no |
| no `package.json`, `index.js` present | ok | `index.js` | yes |
| malformed `package.json` | **fails** | — | no |
| `main` without file extension | ok | the extension-probed file | yes |
| `main` → nested path (`./dist/register.cjs`) | ok | the nested file | yes |
| `main` → a directory containing `index.js` | ok | `<dir>/index.js` | yes |
| `main` → a path outside the root (`../../x.js`) | ok | the outside file | yes |
| `register.js` **and** `package.json` `main` | ok | the **`main`** target, not `register.js` | yes |

Identical across all four Node.js versions, with one exception: a `main` pointing at an **ESM** file (`"type":
"module"`) fails on 18.20.8 and succeeds on 20.20.2, 22.23.4 and 24.22.2, which support `require(esm)`.

These results are why [directory require is rejected](#why-not-directory-require):

- **A root containing only `register.js` and no `package.json` cannot be directory-required at all** —
  `LOAD_AS_DIRECTORY` falls back to `index.js`, never to `register.js`. The well-known filename is reachable only as an
  explicit file path.
- **Every failure mode aborts the host application**, exit code 1, before its first line. There is no graceful
  degradation to fall back on.
- **`main` and `exports` select different files**, and **`register.js` and `main` select different files**. Resolution
  is not confluent, so any rule admitting more than one mechanism must also say which wins.
- **Node.js resolves `main` far more liberally than an injector should.** Extensionless values, directory values, and
  values escaping the root all resolve.

### The packaged distribution root

```
/usr/lib/opentelemetry/nodejs/
├── node_modules/
├── package-lock.json
└── package.json      ← generated by npm, declares only "dependencies"
```

The npm-generated manifest declares **no `main`**, and there is no `index.js`. Consequently:

| Invocation | Result |
| --- | --- |
| `--require /usr/lib/opentelemetry/nodejs/register.js` | instruments; OpenTelemetry API global present |
| `--require /usr/lib/opentelemetry/nodejs` (as packaged) | **`Cannot find module`, exit 1, application never starts** |
| `--require /usr/lib/opentelemetry/nodejs` with `main` added | instruments; OpenTelemetry API global present |
| `--require /usr/lib/opentelemetry/nodejs` with `exports` only | **`Cannot find module`, exit 1** |

An implementation that keyed on the presence of `package.json` would break every Node.js application on a host
carrying today's packaged distribution. The fixed-filename rule is unaffected by any of it.

### Hook versus library

Pointed at an `@opentelemetry/auto-instrumentations-node` installation, a `main`-based rule resolves to the package's
`main`, `./dist/index.cjs`:

| Required file | Application runs | OpenTelemetry API global | Instrumented |
| --- | --- | --- | --- |
| `dist/index.cjs` (the package `main`) | yes | **absent** | **no** |
| `dist/register.cjs` (the `./register` export) | yes | present | yes |

Both exit 0 and print nothing alarming. The first silently produces no telemetry — the failure mode a contract should
never make reachable, and the decisive argument against resolving entry points through `main`.

### Entry point module format

| `register.js` | Node.js 18 | Node.js 20.19+ / 22+ |
| --- | --- | --- |
| CommonJS | loads | loads |
| ESM | **aborts the app** | loads (`require(esm)`) |
| ESM with top-level await | **aborts the app** | **aborts the app** (`ERR_REQUIRE_ASYNC_MODULE`) |

### Upstream register hook: stream usage

`@opentelemetry/auto-instrumentations-node@0.82.0`, required directly, with the application's own output excluded:

| `OTEL_LOG_LEVEL` | lines on stdout | lines on stderr |
| --- | --- | --- |
| `info` (default) | 3 | 1 |
| `debug` | 222 | 1 |

The first stdout line is `OpenTelemetry automatic instrumentation started successfully`.

Instrumentation was confirmed loaded throughout by the presence of the OpenTelemetry API global
(`globalThis[Symbol.for('opentelemetry.js.api.1')]`, carrying `version,context,propagation,metrics,logs`), absent in
the uninstrumented control run.
