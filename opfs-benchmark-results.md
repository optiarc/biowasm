# OPFS Benchmark Findings

This file records manual benchmark results from the host benchmark page:

- [opfs-bench-host.html](/home/lars/git/biowasm/tools/aioli/src/src/examples/opfs-bench-host.html)
- [opfs-bench-host.js](/home/lars/git/biowasm/tools/aioli/src/src/examples/opfs-bench-host.js)

## Small dataset

Input set:

- `small-ref.fa`
- `small-reads.fq.gz`
- `small-sorted.bam`
- `small-unsorted.bam`

Observed outcome:

- All four benchmark cases passed.
- `minimap2` `/opfs` and `/input` performance was effectively the same on this dataset.
- `samtools view` was slightly faster from `/input/...` than from `/opfs/...`.
- JS heap deltas stayed very small relative to input and output sizes.

## Medium dataset

Input sizes:

- `minimap2`: `83,492,425` bytes
- `samtools view`: `37,384,456` bytes

Results:

- `minimap2` `/opfs` input:
  - `509.5s`
  - output `370.7 MB`
- `minimap2` `/input` input:
  - `601.4s`
  - output `370.7 MB`
- `samtools view` `/opfs` input:
  - `13.1s`
  - output `235.9 MB`
- `samtools view` `/input` input:
  - `13.2s`
  - output `235.9 MB`

Observed outcome:

- All four benchmark cases passed.
- `/opfs/...` was materially faster than `/input/...` for `minimap2` on the medium dataset.
- `/opfs/...` and `/input/...` were effectively equivalent for `samtools view`.
- JS heap deltas stayed in the tens to low hundreds of KB while outputs were hundreds of MB, which is consistent with avoiding large JS-side buffering.

## Current interpretation

- Direct OPFS output is working correctly.
- Mounted `/input/...` reads are working correctly.
- For larger `minimap2` workloads, importing to OPFS first may be worthwhile if the same inputs will be reused.
- For one-off `samtools view` runs, `/input/...` appears acceptable without a meaningful performance penalty on the tested datasets.
