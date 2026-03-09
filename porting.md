# Porting biowasm/Aioli to OPFS for Large-File Output

## Goal

Support running `minimap2` in the browser with output written directly to the browser's Origin Private File System (OPFS), without modifying `minimap2` itself.

This document assumes:

- the target browser has full OPFS support
- large output files are expected, so copying results out of an in-memory virtual filesystem is not acceptable
- `minimap2` continues to use normal POSIX-style file I/O (`fopen`, `write`, stdout redirection, etc.)
- the current biowasm/Aioli stack may be changed

## Current State

The current repo is built around an older Emscripten filesystem model:

- shared compiler flags export `FS`, `PROXYFS`, and `WORKERFS`
- tool modules are built with JS filesystem helpers such as `-lworkerfs.js` and `-lproxyfs.js`
- tool examples and docs describe Aioli mounts in terms of the existing virtual filesystem

This is sufficient for transient virtual files and lazy-mounted input files, but not for direct persistent writes of very large outputs into OPFS.

## Target State

The desired architecture is:

1. Aioli initializes tool modules with WasmFS enabled.
2. Aioli mounts an OPFS-backed directory inside each module, for example `/opfs`.
3. `minimap2` writes directly to paths under `/opfs`, such as `/opfs/results/output.sam`.
4. Output remains in browser-managed persistent storage and does not need to be copied out of MEMFS after the command finishes.

## Non-Goals

- changing `minimap2` source code
- redesigning minimap2 command-line semantics
- preserving compatibility with every existing filesystem backend during the first migration step

## Work Areas

### 1. Upgrade the Emscripten Toolchain

The repo documents `Emscripten 2.0.25`, which predates the current WasmFS/OPFS direction.

Required work:

- choose and pin a modern Emscripten version with WasmFS + OPFS support
- update local dev container and build instructions
- verify that the generated JS/wasm output remains compatible with Aioli's module loading model
- revalidate all shared post-processing in `bin/compile.sh` against the new Emscripten output

Risks:

- generated glue code may differ enough that the current `sed` post-processing is no longer valid
- exported runtime methods and startup hooks may change

### 2. Replace the Legacy Filesystem Assumptions

The current shared flags are centered on the old JS FS stack. OPFS support should be treated as a WasmFS migration, not a small add-on flag.

Required work:

- update shared compiler flags in `bin/shared.sh`
- remove assumptions that `WORKERFS` and `PROXYFS` are the primary mounting mechanisms
- enable WasmFS for modules that need OPFS-backed output
- include the necessary Emscripten support for the OPFS backend

Open question:

- whether the repo should move all tools to WasmFS at once, or introduce a dual mode where only selected tools use OPFS-capable builds initially

Recommendation:

- start with a dual mode only if required for rollout safety
- otherwise migrate the shared runtime once, because Aioli is the abstraction layer and mixed filesystem models will add complexity quickly

### 3. Add OPFS Mounting to Aioli

This is the main product-facing change.

Aioli currently exposes:

- tool initialization
- command execution
- mounting of local files, blobs, strings, and URLs
- virtual filesystem utilities

Aioli needs a new concept: a persistent OPFS-backed mount.

Required work:

- add Aioli initialization logic that creates or attaches an OPFS backend during module startup
- mount that backend at a stable path, for example `/opfs`
- expose an API for preparing output directories, for example `await CLI.mkdir("/opfs/results")`
- ensure command execution can target OPFS paths directly
- expose APIs for listing, stat'ing, reading, deleting, and exporting OPFS-backed files

Possible API additions:

```js
const CLI = await new Aioli(["minimap2/2.22"], {
  filesystem: "opfs"
});

await CLI.mkdir("/opfs/results");
await CLI.exec("minimap2", [
  "-a",
  "/inputs/ref.fa",
  "/inputs/reads.fa",
  "-o",
  "/opfs/results/aln.sam"
]);
```

Alternative API:

```js
await CLI.mountOpfs({ path: "/opfs" });
```

Recommendation:

- prefer implicit OPFS initialization behind a config flag over a separate mount call
- keep the mounted path fixed and predictable

Reasoning:

- large-file workflows are easier to document if output paths are stable
- avoiding optional mount order reduces runtime edge cases

### 4. Update Aioli's File Abstractions

Aioli's current mount behavior is optimized around:

- browser `File` objects
- remote URLs
- ephemeral virtual paths

For large-file workflows, Aioli should distinguish:

- read-only mounted inputs
- transient scratch space
- persistent OPFS output

Required work:

- define path conventions, for example:
  - `/inputs` for mounted user files or URLs
  - `/tmp` for transient tool scratch data
  - `/opfs` for persistent outputs
- ensure utility methods like `ls`, `stat`, `unlink`, and download/export work correctly across these path classes
- document persistence behavior clearly so users know which paths survive page reloads

### 5. Define the `minimap2` Execution Contract

Assuming `minimap2` remains unmodified, the runtime contract should be:

- input files may come from mounted local files, URLs, or copied/prepared OPFS inputs
- output files are written to explicit file paths under `/opfs`
- stdout should be reserved for small textual outputs or debugging, not primary large alignment output

Required work:

- document recommended `minimap2` invocation patterns
- prefer `-o /opfs/...` over shell redirection where possible
- verify that output creation, truncation, append semantics, and error reporting behave as expected on OPFS-backed paths

Recommended command pattern:

```bash
minimap2 -a /inputs/ref.fa /inputs/reads.fq -o /opfs/results/aln.sam
```

### 6. Handle Large Input Files Explicitly

Large output is the immediate goal, but in practice large input handling matters too.

Questions to resolve:

- should large user-selected inputs stay as browser file mounts, or be staged into OPFS first?
- do workflows need resumable preprocessing across reloads?
- do downstream tools need to consume the same large files later without asking the user to select them again?

Suggested phased approach:

1. Phase 1: direct OPFS output only
2. Phase 2: optional import of large inputs into OPFS
3. Phase 3: persistent multi-step workflows entirely inside OPFS

This keeps the first migration scoped and avoids coupling output persistence to input persistence.

### 7. Update Tool Build Recipes as Needed

If `minimap2` itself stays unchanged, most of the work is still in the shared build/runtime layer. However, tool recipes may still need adjustments.

Required work:

- verify that `tools/minimap2/compile.sh` still builds under the new Emscripten version
- verify that the existing `v2.22.patch` is still sufficient
- confirm whether any compile flags related to filesystem support need to move from shared flags into per-tool flags
- rebuild both `minimap2` and `minimap2-simd`

Expected result:

- no algorithmic patch to minimap2
- possible build-flag changes only

### 8. Revisit Shared Glue-Code Patches

The current shared build script rewrites generated glue code with `sed`.

Required work:

- verify whether those rewrites are still needed on the newer toolchain
- remove them if obsolete
- if still needed, replace brittle text replacements with a version-gated or generated approach where possible

This matters because filesystem internals often change across Emscripten releases.

### 9. Update Documentation and Examples

The user-facing docs currently explain the virtual filesystem model. That will be incomplete once persistent OPFS output is supported.

Required work:

- add an example showing `minimap2` writing to `/opfs/results/aln.sam`
- document the persistence model:
  - what survives reload
  - what is origin-scoped
  - how to clear old outputs
- document recommended usage for large outputs
- explain that OPFS paths are browser-private and not directly visible as host OS files

### 10. Add Verification for Large-File Workflows

Small functional tests are not enough here. The key property is avoiding large in-memory copies.

Required work:

- add integration tests that write outputs to `/opfs`
- verify files persist across module reinitialization
- verify files persist across page reload if that is part of the product contract
- measure peak memory during a representative large-file write
- test cleanup flows for deleting large OPFS outputs

Success criteria:

- a large output file can be produced without copying the full file through JS memory after command completion
- a second command can consume the OPFS output directly

## Proposed Rollout Plan

### Phase 1: Runtime Spike

- upgrade one branch of the build stack to a modern Emscripten
- enable WasmFS + OPFS for a single tool path
- prove that a small demo tool can write directly into `/opfs`

Exit criterion:

- persistent file creation works end-to-end in the browser

### Phase 2: Aioli Integration

- add stable OPFS initialization to Aioli
- expose the minimal public API
- add one end-to-end `minimap2` example

Exit criterion:

- `minimap2 -o /opfs/...` works in a browser example without tool source changes

### Phase 3: Hardening

- validate large-file behavior
- test persistence and cleanup
- update docs
- decide whether other tools should adopt the same output model

Exit criterion:

- the workflow is stable enough to become the recommended pattern for large outputs

## Suggested Initial Deliverables

1. A build branch with modern Emscripten and WasmFS enabled.
2. A minimal Aioli OPFS API.
3. A `minimap2` browser example writing directly to `/opfs/results/aln.sam`.
4. A persistence test covering module reinitialization.
5. Updated docs describing the new output path model.

## Summary

If the browser fully supports OPFS and `minimap2` stays unmodified, the migration is primarily an Aioli and Emscripten runtime port, not a `minimap2` port.

The main engineering tasks are:

- move from the legacy filesystem model to WasmFS-capable builds
- add OPFS-backed mounts in Aioli
- define stable path conventions for persistent output
- verify that large outputs are written directly to persistent storage without copy-out through JS memory
