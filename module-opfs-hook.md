# Module OPFS Hook Contract

## Purpose

This document defines the module/runtime hook that Aioli's future `opfsBackend: "direct"` mode expects biowasm-built modules to expose.

Today, Aioli uses a staged backend that copies data between browser OPFS and the wasm filesystem. The goal of the direct backend is to avoid those copies and let tools read and write `/opfs/...` paths directly.

## Required Module Hook

Each OPFS-capable biowasm module should expose a function with the following shape:

```js
await module.mountOpfs({
  root: "/opfs",
  FS: module.FS,
});
```

Current status:

- [bin/compile.sh](/home/lars/git/biowasm/bin/compile.sh#L53) now injects a default `mountOpfs()` stub into generated module wrappers.
- That stub currently throws a clear error explaining that the module must be rebuilt with direct OPFS support.
- This means runtimes can now probe for `module.mountOpfs` consistently even before the direct backend exists.
- Generated wrappers also expose:
  - `module.biowasmCapabilities.mountOpfs`
  - `module.biowasmCapabilities.mountOpfsSource`
- This lets runtimes distinguish real support from a stub and understand where the capability signal came from.

## Expected Behavior

`module.mountOpfs(...)` should:

- prepare the module filesystem so the path in `root` exists
- mount or attach an OPFS-backed filesystem at that path
- make ordinary libc/POSIX file operations against that path work for the tool
- return only after the mount is ready for command execution

In other words, after:

```js
await module.mountOpfs({ root: "/opfs", FS: module.FS });
```

this should work without any extra copy step:

```bash
minimap2 -a -o /opfs/results/aln.sam ...
```

## Non-Goals

This hook should not:

- copy existing files from browser OPFS into MEMFS as a staging step
- copy output files back into browser OPFS after the command
- expose a second caller-facing path different from `/opfs`

Those are properties of the current staged backend, not the desired direct backend.

## Responsibilities Split

### Aioli responsibilities

- choose the backend mode (`staged` vs future `direct`)
- call `module.mountOpfs(...)` during module initialization for the direct backend
- keep `/opfs/...` as the caller-facing path contract

### biowasm module responsibilities

- compile modules with the runtime/filesystem support needed for direct OPFS mounts
- expose `module.mountOpfs(...)`
- ensure the mounted filesystem works with the module's existing `FS` and `callMain` flow

## Build-System Implications

The current shared flags in [bin/shared.sh](/home/lars/git/biowasm/bin/shared.sh#L1) are still based on the legacy JS filesystem stack:

- `FS`
- `PROXYFS`
- `WORKERFS`
- `-lworkerfs.js`
- `-lproxyfs.js`

The direct backend will require a different runtime path, likely centered on newer Emscripten filesystem support rather than the current PROXYFS/WORKERFS setup.

## Acceptance Criteria

The module hook is good enough for the first direct-backend implementation when all of the following are true:

1. Aioli can call `module.mountOpfs({ root: "/opfs", FS: module.FS })`.
2. A tool can write directly to `/opfs/...` without using `opfsStage()` or `opfsFlush()`.
3. The resulting file persists in browser OPFS.
4. No full-file `readFile()` / `writeFile()` copy is needed in JS to make that happen.

## First Consumer

The first direct-backend acceptance test should be the existing `minimap2` browser workflow:

```bash
minimap2 -a -o /opfs/results/aln.sam /minimap2/MT-human.fa /minimap2/MT-orang.fa
```

That should replace the current staged flow that writes to the wasm filesystem and then flushes into browser OPFS afterward.
