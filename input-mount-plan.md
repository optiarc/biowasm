# Input Mount Plan

This document describes the smallest path to support user-provided input files "in place" without copying them into OPFS first.

## Goal

Support workflows like:

```bash
samtools view -o /opfs/results/out.sam /input/user.bam
minimap2 -a -o /opfs/results/aln.sam /input/ref.fa /input/reads.fq
```

with these properties:

- user-provided inputs are read-only
- inputs are not copied into OPFS before execution
- outputs still write directly to `/opfs/...`

## Important current state

Aioli already has a partial version of this idea.

In [tools/aioli/src/src/worker.js](/home/lars/git/biowasm/tools/aioli/src/src/worker.js), `mount(files)`:

- accepts browser `File` objects, `Blob`s, strings, and URLs
- mounts `File`/`Blob` inputs through Emscripten `WORKERFS`
- creates symlinks into the shared writable data directory

That means read-only mounted user input already exists conceptually. What is missing is a stable public path contract and tighter integration with the newer `/opfs/...` output model.

## Proposed target contract

Use two top-level path classes:

- `/input/...`
  - read-only, backed by browser `File` objects
- `/opfs/...`
  - writable, backed by persistent OPFS

Example:

```js
await CLI.mountInputs({
  "/input/user.bam": file,
  "/input/ref.fa": refFile,
  "/input/reads.fq": readsFile
});
```

Then tools run unchanged:

```js
await CLI.exec("samtools view -o /opfs/results/out.sam /input/user.bam");
await CLI.exec("minimap2 -a -o /opfs/results/aln.sam /input/ref.fa /input/reads.fq");
```

## Concrete file touch points

The smallest first implementation should touch these files:

- [tools/aioli/src/src/main.js](/home/lars/git/biowasm/tools/aioli/src/src/main.js)
  - add a default public input root such as `dirInput: "/input"`
  - optionally add an internal mounted-input root if different from the public path

- [tools/aioli/src/src/worker.js](/home/lars/git/biowasm/tools/aioli/src/src/worker.js)
  - extend `mount(files)` or add a new `mountInputs(...)`
  - add `/input/...` handling to `_resolveFsPath(...)`
  - add `/input/...` handling to `_publicFsPath(...)`
  - ensure `_fileop(...)` works for mounted input paths
  - ensure write-oriented operations reject `/input/...` clearly

- [tools/aioli/src/tests/...](/home/lars/git/biowasm/tools/aioli/src/tests)
  - add focused `/input` behavior tests
  - add one `minimap2` input-mounted workflow test
  - add one `samtools` input-mounted workflow test

- [tools/aioli/src/src/examples/...](/home/lars/git/biowasm/tools/aioli/src/src/examples)
  - add a small manual browser example for mounted `/input/...` plus `/opfs/...` output

## Suggested implementation sketch

### Config

Add to [tools/aioli/src/src/main.js](/home/lars/git/biowasm/tools/aioli/src/src/main.js):

```js
dirInput: "/input"
```

### Worker state

Track mounted input metadata separately from generic mounted files if needed:

```js
inputFiles: []
```

This is optional, but it makes `listInputs()` and `unmountInputs()` easier to implement cleanly.

### API surface

Recommended additions:

```js
await CLI.mountInputs(filesOrMap)
await CLI.listInputs()
await CLI.unmountInputs()
```

The first implementation can wrap the existing `mount(files)` logic instead of replacing it.

### Path handling

Update [tools/aioli/src/src/worker.js](/home/lars/git/biowasm/tools/aioli/src/src/worker.js):

- `_resolveFsPath(path)`
  - map `/input/...` to the mounted read-only location
- `_publicFsPath(path)`
  - map mounted internal paths back to `/input/...`

This keeps the caller-facing contract aligned with what was already done for `/opfs/...`.

### Read-only enforcement

For the first version, explicit rejection is enough:

- `write(...)` on `/input/...` throws
- `mkdir(...)` on `/input/...` throws
- `rm`, `unlink`, `rename`, `rmdir` on `/input/...` throw

No special persistence is needed because `/input` is intentionally ephemeral and read-only.

## Smallest implementation path

### Step 1. Keep using `WORKERFS` for user files

Do not build a new read backend first.

Reuse the existing `mount(files)` implementation and make it the engine behind a new `/input` API.

Why:

- `WORKERFS` already exposes browser `File` objects without a full up-front copy into OPFS
- it already matches the needed read-only semantics
- it is a much smaller change than designing a new file backend from scratch

### Step 2. Add a stable public `/input` path

Today, mounted files are effectively exposed through the shared data area and symlinks.

Add an explicit public contract:

- mounted user files appear under `/input/...`
- tools should treat `/input/...` as read-only

Internally, the first implementation can still route this through the current mounted/shared directories.

### Step 3. Add an explicit Aioli API

Add one high-level method:

```js
await CLI.mountInputs(filesOrMap)
```

Recommended accepted shapes:

```js
await CLI.mountInputs([file1, file2]);
await CLI.mountInputs({
  "/input/user.bam": file,
  "/input/ref.fa": refFile
});
```

Also add:

- `listInputs()`
- `unmountInputs()`

### Step 4. Make path resolution understand `/input/...`

The worker already translates `/opfs/...` public paths.

Add equivalent path handling for:

- `exec()` arguments
- `cat`, `ls`, `download`, `read`
- any helper that should work on input files

### Step 5. Preserve strict read-only behavior

Do not allow tool writes to `/input/...`.

Expected behavior:

- reads succeed
- writes fail clearly
- rename/unlink/mkdir/rmdir on `/input/...` fail clearly

That keeps the contract simple and prevents confusion between ephemeral mounted inputs and persistent OPFS outputs.

## Why this is likely enough

For the user’s main question, the answer is probably already "mostly yes" in architecture:

- browser `File` -> `WORKERFS` mount
- tool reads from mounted path
- output goes to `/opfs/...`

The missing work is mostly API cleanup and explicit path modeling, not deep tool changes.

## Current implemented status

The first practical `/input` path is now implemented and browser-tested in Aioli.

What exists now:

- `dirInput: "/input"` in [tools/aioli/src/src/main.js](/home/lars/git/biowasm/tools/aioli/src/src/main.js)
- `mountInputs()`, `listInputs()`, and `unmountInputs()` in [tools/aioli/src/src/worker.js](/home/lars/git/biowasm/tools/aioli/src/src/worker.js)
- explicit write rejection for `/input/...`
- browser harnesses and Cypress specs for:
  - API-level `/input` mount behavior
  - `minimap2` reading `/input/...` and writing to `/opfs/...`
  - `samtools view` reading `/input/...` and writing to `/opfs/...`

There is also now a first cross-source comparison harness:

- [input-vs-opfs-bench.html](/home/lars/git/biowasm/tools/aioli/src/src/examples/input-vs-opfs-bench.html)
- [input-vs-opfs-bench.js](/home/lars/git/biowasm/tools/aioli/src/src/examples/input-vs-opfs-bench.js)
- [test_input_vs_opfs_bench.cy.js](/home/lars/git/biowasm/tools/aioli/src/tests/test_input_vs_opfs_bench.cy.js)

That harness compares reading the same inputs from:

- `/opfs/...`
- `/input/...`

for:

- `minimap2`
- `samtools view`

and is now browser-tested.

There is also a manual large-file companion harness for host-selected files:

- [opfs-bench-host.html](/home/lars/git/biowasm/tools/aioli/src/src/examples/opfs-bench-host.html)
- [opfs-bench-host.js](/home/lars/git/biowasm/tools/aioli/src/src/examples/opfs-bench-host.js)

That page is designed for the current larger dataset available on this machine under `/opt/lucemics/data` and compares `/input/...` versus `/opfs/...` reads for:

- `minimap2`
- `samtools view`

It relies on direct OPFS `Blob` writes handling large files incrementally rather than via a whole-file `arrayBuffer()` conversion.

## Likely supported workflows

These should be the first targets:

- `minimap2` reading FASTA/FASTQ from `/input/...`
- `samtools view -o /opfs/... /input/user.bam`
- `samtools fastq -o /opfs/... /input/user.bam`
- similar explicit-output pipelines

## Risks and limits

### 1. Read-only only

This design is intentionally not for writable user files.

### 2. Browser-session scoped

Mounted input files are ephemeral browser objects, not persisted storage.

### 3. Random-access performance

Formats like BAM may do many seeks. `WORKERFS` can still work, but performance should be measured for:

- large BAM reads
- indexed access
- repeated passes over the same file

For repeated workflows, copying once to OPFS may still be better than repeated host-file reads.

### 4. Sidecar/index expectations

If a tool reads `/input/foo.bam` and creates `foo.bam.bai`, that output must go somewhere writable.

The recommended contract is:

- inputs stay under `/input/...`
- outputs go to explicit `/opfs/...` paths

Do not rely on implicit writable sidecars beside `/input/...` files.

## Recommended tests

### 1. API-level input mount

- mount a browser `File`
- verify `ls("/input")`
- verify `cat("/input/file.txt")`
- verify writes to `/input/...` fail

### 2. minimap2 input-in-place

```bash
minimap2 -a -o /opfs/results/aln.sam /input/ref.fa /input/reads.fq
```

Validate:

- no input copy to OPFS required
- output appears in OPFS

### 3. samtools BAM input-in-place

```bash
samtools view -o /opfs/results/out.sam /input/user.bam
```

Validate:

- tool can read the mounted BAM
- output writes to OPFS

### 4. Repeated-run memory/performance

Use the benchmark harness later to compare:

- mounted `/input/...` reads
- copied-to-OPFS reads

for the same large BAM/FASTQ inputs.

## First-pass repo test matrix

### Unit/integration level

Add to [tools/aioli/src/tests/test_inputs.cy.js](/home/lars/git/biowasm/tools/aioli/src/tests/test_inputs.cy.js) or a new focused file:

- mount one small text file at `/input/example.txt`
- verify `ls("/input")`
- verify `cat("/input/example.txt")`
- verify `download("/input/example.txt")`
- verify `write("/input/example.txt", ...)` fails

### Minimap2 workflow

New focused spec:

- mount `large-ref.fa` and `large-reads.fq` or smaller test FASTA/FASTQ inputs at `/input/...`
- run:

```bash
minimap2 -a -o /opfs/results/aln.sam /input/ref.fa /input/reads.fq
```

- verify output exists in OPFS

### Samtools workflow

New focused spec:

- mount a user BAM at `/input/user.bam`
- run:

```bash
samtools view -o /opfs/results/out.sam /input/user.bam
```

- verify output exists in OPFS

### Negative path

Verify that:

- `samtools index /input/user.bam` fails if it tries to create a sidecar beside the input file
- the documented workaround is an explicit OPFS output path workflow

This is important because it makes the `/input` contract explicit and prevents accidental write expectations.

## Rollout sequence

1. Add `mountInputs()` API on top of the existing mount mechanism.
2. Expose a stable public `/input/...` path.
3. Add focused tests for `/input` read-only behavior.
4. Add one `minimap2` `/input` example.
5. Add one `samtools` `/input` example.
6. Benchmark large-file reads versus copied-to-OPFS reads.

## Recommended first milestone

The best first milestone is intentionally narrow:

1. `mountInputs()` exists
2. `/input/...` paths resolve and are readable
3. `minimap2` can read `/input/ref.fa` and `/input/reads.fq`
4. output writes to `/opfs/...`

If that works, `samtools view` should be the next consumer.

## Current implementation status

The first narrow `/input` milestone is now partially implemented in Aioli:

- `dirInput: "/input"` exists in [tools/aioli/src/src/main.js](/home/lars/git/biowasm/tools/aioli/src/src/main.js)
- `mountInputs()`, `listInputs()`, and `unmountInputs()` exist in [tools/aioli/src/src/worker.js](/home/lars/git/biowasm/tools/aioli/src/src/worker.js)
- mounted input files are exposed read-only under `/input/...`
- explicit writes to `/input/...` are rejected

Browser-tested coverage exists for:

- mounting a browser `File` under `/input/...`
- listing `/input`
- reading a mounted `/input/...` file
- rejecting writes to `/input/...`

Implemented test/example files:

- [tools/aioli/src/src/examples/input-mount-test.html](/home/lars/git/biowasm/tools/aioli/src/src/examples/input-mount-test.html)
- [tools/aioli/src/src/examples/input-mount-test.js](/home/lars/git/biowasm/tools/aioli/src/src/examples/input-mount-test.js)
- [tools/aioli/src/tests/test_input_mount.cy.js](/home/lars/git/biowasm/tools/aioli/src/tests/test_input_mount.cy.js)

Still not implemented:

- broader `minimap2` `/input` coverage beyond the first mounted FASTA harness
- broader `samtools` `/input` coverage beyond the first mounted BAM harness
- performance comparisons between `/input/...` reads and copied-to-OPFS reads

The first real tool consumer is now browser-tested:

- [tools/aioli/src/src/examples/minimap2-input-opfs-test.html](/home/lars/git/biowasm/tools/aioli/src/src/examples/minimap2-input-opfs-test.html)
- [tools/aioli/src/src/examples/minimap2-input-opfs-test.js](/home/lars/git/biowasm/tools/aioli/src/src/examples/minimap2-input-opfs-test.js)
- [tools/aioli/src/tests/test_minimap2_input_opfs.cy.js](/home/lars/git/biowasm/tools/aioli/src/tests/test_minimap2_input_opfs.cy.js)

That harness verifies:

- bundled FASTA data can be re-mounted as browser `File` inputs under `/input/...`
- `minimap2` can read `/input/MT-human.fa` and `/input/MT-orang.fa`
- output writes directly to `/opfs/results/aln.sam`

The first `samtools` `/input` consumer is also now browser-tested:

- [tools/aioli/src/src/examples/samtools-input-opfs-test.html](/home/lars/git/biowasm/tools/aioli/src/src/examples/samtools-input-opfs-test.html)
- [tools/aioli/src/src/examples/samtools-input-opfs-test.js](/home/lars/git/biowasm/tools/aioli/src/src/examples/samtools-input-opfs-test.js)
- [tools/aioli/src/tests/test_samtools_input_opfs.cy.js](/home/lars/git/biowasm/tools/aioli/src/tests/test_samtools_input_opfs.cy.js)

That harness verifies:

- a BAM file can be mounted under `/input/user.bam`
- `samtools view` can read `/input/user.bam`
- output writes to `/opfs/results/out.sam`

## Recommendation

Do not start by building a brand-new custom input filesystem.

Start by formalizing what Aioli already has:

- `WORKERFS` mounted user files
- explicit public `/input/...` paths
- direct `/opfs/...` outputs

That is the shortest path to a practical "read in place, write to OPFS" workflow without modifying `samtools` or `minimap2`.
