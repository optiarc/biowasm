# samtools OPFS Support Matrix

## Purpose

This document summarizes what level of `samtools` support is realistic under the current Aioli direct OPFS model, what is likely to fail, and what would be required to make `samtools` behave like a normal tool writing directly to a persistent filesystem.

Assumption:

- `samtools` itself remains unmodified
- the work happens in Aioli and the runtime filesystem layer

## Current Model

Today, Aioli's experimental `opfsBackend: "direct"` mounts an Aioli-managed filesystem at `/opfs` inside the current Emscripten runtime and backs explicit `/opfs/...` files with OPFS sync access handles in the worker.

This is already enough for tools that:

- write to an explicitly named output file
- read that same file back later
- do not require broad directory mutation semantics during execution

This is not yet a full persistent filesystem implementation. It is closer to:

- explicit file path preparation
- direct streaming reads and writes for those files
- partial directory handling

It is not yet a complete implementation of arbitrary runtime-created files, renames, or temp-file-heavy workflows.

## Support Matrix

Legend:

- `Works now`: should work with the current direct backend
- `Likely with small Aioli work`: probably possible without touching `samtools`, but needs backend improvements
- `Needs fuller backend`: depends on more complete filesystem semantics

| Workflow | Status | Why |
| --- | --- | --- |
| `samtools view input.bam > stdout` | Works now | No OPFS output file involved |
| `samtools view -o /opfs/out.sam input.bam` | Works now | Explicit output path can be prepared before execution |
| `samtools fastq -o /opfs/out.fastq input.bam` | Works now | Same pattern as explicit `-o` output |
| `samtools faidx ref.fa chr1:1-100 > stdout` | Works now | Pure stdout workflow |
| `samtools faidx ref.fa -o /opfs/out.fa region` | Works now | Explicit output path |
| `samtools view /opfs/out.sam` | Works now | Direct backend can reopen prepared files from OPFS |
| `samtools view /opfs/input.bam -o /opfs/out.sam` | Works now | Direct input staging plus explicit OPFS output is browser-tested |
| `samtools sort -o /opfs/out.bam input.bam` | Works now | Explicit sorted output is browser-tested under the current direct backend |
| `samtools merge -o /opfs/out.bam a.bam b.bam` | Likely with small Aioli work | Usually explicit output, but should be tested for internal temp behavior |
| `samtools index /opfs/file.bam` | Works now with targeted Aioli support | Current backend prebinds the `.bai` sidecar path specifically for samtools |
| `samtools faidx /opfs/ref.fa` | Works now with targeted Aioli support | Current backend prebinds the `.fai` sidecar path specifically for samtools |
| `samtools dict /opfs/ref.fa -o /opfs/ref.dict` | Works now | Explicit output path |
| `samtools calmd`, `fixmate`, `markdup`, `reheader` with explicit `-o /opfs/...` | Likely with small Aioli work | Depends on subcommand-specific file handling |
| Any command that writes temporary files under `/opfs` and renames them into place | Needs fuller backend | Current direct backend does not yet persist rename semantics to real OPFS |
| Any command that expects directory scans, deletion, cleanup, and sidecar discovery to behave like a normal disk filesystem | Needs fuller backend | Directory and mutation semantics are still partial |

## Why `sort` and `index` Are Harder

### `samtools sort`

`samtools sort` was expected to be harder than `view -o` because the final output path is only part of the workflow.

Typical risks:

- temporary chunks may be created before the final file exists
- temp filenames may be generated internally rather than passed explicitly by the caller
- temp files may be deleted after merging
- the final output may be assembled through rename-like behavior rather than a single straightforward write stream

Why that matters for the current backend:

- the current direct backend is strongest when Aioli knows the path up front and can prepare it before execution
- temp files created dynamically by the tool are not necessarily pre-registered with OPFS sync handles
- delete and rename semantics are not yet fully mirrored into persistent OPFS state

Observed current status:

- `samtools sort -o /opfs/out.bam input.bam` works in the current browser-tested harness

Remaining caveat:

- broader temp-file-heavy or merge-style workflows should still be tested before treating sort-like behavior as generally solved

### `samtools index`

`samtools index` is harder because it usually creates a derived file whose name is implicit.

Examples:

- `/opfs/file.bam` -> `/opfs/file.bam.bai`
- or sometimes a CSI sidecar instead of BAI depending on usage

Why that matters:

- there is no explicit `-o /opfs/file.bam.bai` path for Aioli to prepare in advance in the common case
- the filesystem layer must support the tool creating new files on its own
- sidecar lookup later depends on directory and existence semantics working correctly

Current status:

- `samtools index /opfs/file.bam` and `samtools faidx /opfs/ref.fa` now work through targeted Aioli-side sidecar preparation

Remaining limitation:

- this is still command-specific support, not a fully generic implicit-file-creation solution

## What Needs To Be Added for Broader `samtools` Support

To support `samtools` more generally without patching `samtools`, the runtime filesystem backend needs to behave more like a normal persistent filesystem.

### 1. General file creation under `/opfs`

The backend must support tool-driven file creation without prior registration by Aioli.

That means:

- open with create flags should allocate a real OPFS-backed file automatically
- not only for predeclared output paths
- also for tool-generated sidecars and temp files

### 2. Persistent rename semantics

The backend must support rename-like workflows correctly.

That includes:

- file-to-file rename in the same directory
- move into an existing directory
- replace semantics when the destination exists, if needed by the tool

Without this, temp-file-to-final-file workflows remain fragile.

### 3. Persistent unlink and cleanup semantics

The backend must persist deletion to real OPFS, not only in in-memory node state.

That means:

- `unlink`
- `rmdir`
- recursive cleanup where appropriate

This matters for `sort`, temp file cleanup, and any subcommand that removes intermediates.

### 4. More complete directory support

The backend must provide reliable:

- `lookup`
- `readdir`
- `stat`
- existence checks
- sidecar discovery in directories

This matters for later reads of `.bai`, `.csi`, `.fai`, and similar files.

### 5. Full open-mode coverage

The backend should correctly support the patterns used by htslib/samtools:

- create
- truncate
- append where relevant
- read/write
- seek
- allocate or grow

The current backend already covers part of this, but it should be validated against BAM/CRAM-sized workflows.

### 6. Temp-file strategy

A practical runtime design should reserve a predictable temp area, for example:

- `/opfs/tmp`

Then:

- allow subcommands to use it explicitly when flags exist
- ensure temp file churn there is cheap and cleanup is reliable

For commands that do not expose temp path flags, the backend still needs generic create/delete support.

## More Complete Runtime Backend Outline

There are two realistic directions.

### Option A: extend the current Aioli legacy-FS direct backend

This is the smaller incremental path.

Work needed:

1. Intercept generic file creation in the mounted `/opfs` filesystem, not only predeclared files or known sidecars.
2. Back newly created files with OPFS sync access handles automatically.
3. Implement rename by moving entries in both:
   - the Emscripten node tree
   - the underlying OPFS directory tree
4. Implement persistent unlink and rmdir.
5. Improve directory enumeration so sidecar files appear naturally.
6. Add tests for:
   - broader `sort`/`merge` temp-file behavior
   - non-predeclared sidecar-like outputs
   - cleanup of temp files
   - generic implicit file creation under `/opfs`

Advantages:

- no tool rebuild required
- stays close to what already works for `minimap2`

Disadvantages:

- Aioli continues to own a lot of filesystem behavior itself
- correctness burden grows as more POSIX semantics are needed

### Option B: move toward a fuller module/runtime-mounted OPFS filesystem

This is the longer-term cleaner design.

Work needed:

1. Continue the `mountOpfs()` contract already documented in [module-opfs-hook.md](/home/lars/git/biowasm/module-opfs-hook.md#L1).
2. Build modules with a runtime/filesystem model that can mount persistent storage more natively.
3. Let tools interact with `/opfs` through the module/runtime filesystem rather than Aioli emulating more and more behavior in JS.

Advantages:

- better conceptual fit for tools like `samtools`
- less tool-specific special handling in Aioli
- better long-term path for temp files, sidecars, and rename-heavy workflows

Disadvantages:

- larger build/runtime migration
- more upfront complexity

## Recommended Next Steps

1. Promote the current tested workflows into explicitly required support:
   - `samtools view -o /opfs/out.sam`
   - `samtools fastq -o /opfs/out.fastq`
   - `samtools sort -o /opfs/out.bam`
   - `samtools index /opfs/file.bam`
   - `samtools faidx /opfs/ref.fa`
2. Test broader multi-file workflows such as `merge`.
3. Test whether non-predeclared temp or sidecar outputs still fail outside the current targeted samtools support.
4. Based on those results, decide whether:
   - extending the current Aioli direct backend is enough
   - or the project should prioritize the fuller module/runtime-mounted backend

## Bottom Line

`samtools` can already use the OPFS model for more workflows than expected without modifying `samtools` itself.

- explicit output file workflows are working
- `sort -o /opfs/...` is working
- `index` and `faidx` are working through targeted Aioli-side sidecar preparation
- the remaining gap is a generic implicit-file backend, not basic `samtools` OPFS support
