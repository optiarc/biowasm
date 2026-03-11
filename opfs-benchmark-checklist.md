# OPFS Benchmark Checklist

This checklist defines the benchmark cases needed to validate that direct OPFS I/O avoids output-size-proportional RAM growth and copy-out penalties.

## Harness output contract

Each benchmark harness should emit one JSON object per run with these fields:

```json
{
  "case": "minimap2-large-explicit-output",
  "backend": "direct",
  "inputSet": "dev-small",
  "command": "minimap2 -a -o /opfs/results/aln.sam /data/large-ref.fa /data/large-reads.fq",
  "toolMs": 0,
  "totalMs": 0,
  "postCommandMs": 0,
  "outputBytes": 0,
  "jsHeapBefore": 0,
  "jsHeapAfter": 0,
  "jsHeapDelta": 0,
  "uaMemoryBefore": 0,
  "uaMemoryAfter": 0,
  "uaMemoryDelta": 0,
  "stderrSummary": "",
  "passed": false,
  "notes": ""
}
```

Field requirements:

- `toolMs`: extracted from tool stderr when available.
- `totalMs`: outer wall time from immediately before `exec()` to immediately after the file is visible in OPFS.
- `postCommandMs`: time spent after the command returns but before verification completes.
- `outputBytes`: byte size of the OPFS output file.
- `fixtureSource`: identifies whether the run used seeded dev fixtures or generated intermediates.
- `jsHeap*`: from `performance.memory.usedJSHeapSize` when available.
- `uaMemory*`: from `performance.measureUserAgentSpecificMemory()` when available.

Each harness run should also emit run-level metadata including:

- `runLabel`
- `startedAt`
- `finishedAt`
- `status`
- `userAgent`
- `platform`
- `hardwareConcurrency`
- `deviceMemory`

The summary output should also include per-case threshold evaluation for direct-mode runs, including:

- direct memory-limit checks
- direct post-command time-limit checks
- direct-vs-staged timing comparison targets when output size is large enough

## Pass/fail thresholds

Use these thresholds for the direct backend. Compare staged runs separately to confirm that direct is materially better.

### Memory thresholds

- `jsHeapDelta <= max(64 MiB, outputBytes * 0.05)`
- `uaMemoryDelta <= max(96 MiB, outputBytes * 0.10)`
- repeated-run retained memory drift across 3 fresh runs should stay below `128 MiB`

These are intentionally strict enough to catch full-file buffering while still tolerating tool working sets and browser noise.

### Timing thresholds

- `postCommandMs <= max(2000 ms, totalMs * 0.15)`
- direct mode `totalMs` should be at least `20%` faster than staged mode once output size exceeds `1 GiB`
- direct mode `postCommandMs` should be at least `80%` lower than staged mode for the same workload

### Output thresholds

- output file must exist in OPFS immediately after the run
- output byte size must be non-zero
- output validation must pass for the tool-specific check

## Benchmark cases

## Companion read-source comparison harness

In addition to staged-vs-direct output benchmarks, keep a separate comparison harness for input source selection:

- [input-vs-opfs-bench.html](/home/lars/git/biowasm/tools/aioli/src/src/examples/input-vs-opfs-bench.html)
- [input-vs-opfs-bench.js](/home/lars/git/biowasm/tools/aioli/src/src/examples/input-vs-opfs-bench.js)
- [test_input_vs_opfs_bench.cy.js](/home/lars/git/biowasm/tools/aioli/src/tests/test_input_vs_opfs_bench.cy.js)

That harness is for cases where the question is not staged-vs-direct output buffering, but whether a tool can read the same input correctly from:

- `/input/...` mounted browser files
- `/opfs/...` persisted files

Current required comparison coverage:

- `minimap2` reading FASTA from `/input/...` versus `/opfs/...`
- `samtools view` reading BAM from `/input/...` versus `/opfs/...`

This companion harness should stay green before adding larger-file `/input` performance cases.

For the currently available host files under `/opt/lucemics/data`, there is also a manual host-file harness:

- [opfs-bench-host.html](/home/lars/git/biowasm/tools/aioli/src/src/examples/opfs-bench-host.html)
- [opfs-bench-host.js](/home/lars/git/biowasm/tools/aioli/src/src/examples/opfs-bench-host.js)

That page accepts browser-selected files and is intended for the current host dataset:

- `/opt/lucemics/data/large-ref.fa`
- `/opt/lucemics/data/large-reads.fq.gz`
- `/opt/lucemics/data/large-sorted.bam`
- `/opt/lucemics/data/large-unsorted.bam`

It currently runs the first two large read-source comparisons:

- `minimap2` from `/input/...` versus `/opfs/...`
- `samtools view` from `/input/...` versus `/opfs/...`

This harness depends on chunked `Blob` import into direct OPFS writes so large selected files can be copied into `/opfs/...` without whole-file `arrayBuffer()` buffering.

### 1. minimap2 large explicit output

Case id:

```text
minimap2-large-explicit-output
```

Command:

```bash
minimap2 -a -o /opfs/results/aln.sam /data/large-ref.fa /data/large-reads.fq
```

Validation:

- SAM exists at `/results/aln.sam`
- file begins with `@SQ` and includes at least one alignment record

Provide:

- `large-ref.fa`
- `large-reads.fq`

### 2. samtools view large explicit output

Case id:

```text
samtools-view-large-explicit-output
```

Command:

```bash
samtools view -o /opfs/results/out.sam /data/large-sorted.bam
```

Validation:

- SAM exists at `/results/out.sam`
- file begins with `@HD` or `@SQ`

Provide:

- `large-sorted.bam`

### 3. samtools fastq large explicit output

Case id:

```text
samtools-fastq-large-explicit-output
```

Command:

```bash
samtools fastq -0 /opfs/results/out.fastq -o /opfs/results/out.fastq /data/large-sorted.bam
```

Validation:

- FASTQ exists at `/results/out.fastq`
- file contains `@` header lines and sequence lines

Provide:

- `large-sorted.bam`

### 4. samtools sort large explicit output

Case id:

```text
samtools-sort-large-explicit-output
```

Command:

```bash
samtools sort -o /opfs/results/sorted.bam /data/large-unsorted.bam
```

Validation:

- BAM exists at `/results/sorted.bam`
- `samtools view /opfs/results/sorted.bam` succeeds in a follow-up validation step

Provide:

- `large-unsorted.bam`

### 5. samtools index large sidecar

Case id:

```text
samtools-index-large-sidecar
```

Command:

```bash
samtools index /opfs/results/sorted.bam
```

Validation:

- `/results/sorted.bam.bai` exists
- file size is non-zero

Provide:

- `large-sorted.bam`

### 6. samtools faidx large sidecar

Case id:

```text
samtools-faidx-large-sidecar
```

Command:

```bash
samtools faidx /opfs/data/large-ref-indexable.fa
```

Validation:

- `/data/large-ref-indexable.fa.fai` exists
- file size is non-zero

Provide:

- `large-ref-indexable.fa`

### 7. OPFS input to OPFS output roundtrip

Case id:

```text
samtools-opfs-roundtrip-large
```

Command:

```bash
samtools view -o /opfs/results/from-opfs.sam /opfs/data/large-sorted.bam
```

Validation:

- `/results/from-opfs.sam` exists
- SAM contains expected header lines

Provide:

- `large-sorted.bam`

## Input sets

Use these labels in harness output:

- `dev-small`
  - tiny fixtures used for harness wiring and schema validation
- `smoke-medium`
  - around `250 MiB` to `1 GiB` output
- `bench-large`
  - around `5 GiB` output
- `bench-xlarge`
  - around `10 GiB+` output

## Development fixtures

Seed files live under:

- [tools/aioli/src/tests/data/opfs-bench](/home/lars/git/biowasm/tools/aioli/src/tests/data/opfs-bench)

Development harnesses:

- [tools/aioli/src/src/examples/opfs-bench-dev.html](/home/lars/git/biowasm/tools/aioli/src/src/examples/opfs-bench-dev.html)
- [tools/aioli/src/tests/test_opfs_bench_dev.cy.js](/home/lars/git/biowasm/tools/aioli/src/tests/test_opfs_bench_dev.cy.js)

Current fixture status:

- `large-ref.fa`: tiny valid FASTA
- `large-reads.fq`: tiny valid FASTQ
- `large-ref-indexable.fa`: tiny valid FASTA
- `large-unsorted.sam`: tiny valid SAM source
- `large-sorted.sam`: tiny valid SAM source
- `large-unsorted.bam`: placeholder file, replace with a valid BAM before real samtools benchmarks
- `large-sorted.bam`: placeholder file, replace with a valid BAM before real samtools benchmarks

### Replacing with larger real files

Keep the benchmark filenames stable and replace file contents in place:

- `large-ref.fa`
- `large-reads.fq`
- `large-ref-indexable.fa`
- `large-unsorted.bam`
- `large-sorted.bam`

This keeps the existing harness commands unchanged.

After replacement, rerun either:

- the browser harness at [opfs-bench-dev.html](/home/lars/git/biowasm/tools/aioli/src/src/examples/opfs-bench-dev.html)
- or the focused spec at [test_opfs_bench_dev.cy.js](/home/lars/git/biowasm/tools/aioli/src/tests/test_opfs_bench_dev.cy.js)

## Manual development run

Start the Aioli dev server:

```bash
cd /home/lars/git/biowasm/tools/aioli/src
npm run dev
```

Then open:

```text
http://localhost:5173/src/examples/opfs-bench-dev.html
```

The page will:

- run the development-size benchmark cases in staged and direct modes
- print the JSON results to the page
- set `Status` to `PASS` when every development case succeeds

## Automated development run

Run the focused Cypress spec:

```bash
cd /home/lars/git/biowasm/tools/aioli/src
DISPLAY=:99 npx cypress run --browser chromium --spec tests/test_opfs_bench_dev.cy.js
```

The spec writes the latest JSON results to:

- [tools/aioli/src/tests/.artifacts/opfs-bench-dev-results.json](/home/lars/git/biowasm/tools/aioli/src/tests/.artifacts/opfs-bench-dev-results.json)
- [tools/aioli/src/tests/.artifacts/opfs-bench-dev-summary.json](/home/lars/git/biowasm/tools/aioli/src/tests/.artifacts/opfs-bench-dev-summary.json)
- [tools/aioli/src/tests/.artifacts/opfs-bench-dev-metadata.json](/home/lars/git/biowasm/tools/aioli/src/tests/.artifacts/opfs-bench-dev-metadata.json)

## Run procedure

For each case:

1. Clear OPFS.
2. Load fresh Aioli instance.
3. Record memory baseline.
4. Run the case in `staged`.
5. Record metrics and output size.
6. Clear OPFS and reload fresh Aioli instance.
7. Run the same case in `direct`.
8. Record metrics and compare against thresholds.
9. Repeat 3 times for stable numbers.

## Required large files from user

Replace the development fixtures later with:

- `large-ref.fa`: `50 MiB` to `200 MiB` FASTA
- `large-reads.fq`: enough reads to produce `1 GiB+`, ideally `5 GiB+`, SAM output
- `large-unsorted.bam`: `5 GiB+` unsorted BAM
- `large-sorted.bam`: `5 GiB+` sorted BAM
- `large-ref-indexable.fa`: `1 GiB+` FASTA
