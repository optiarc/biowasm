# OPFS Porting History

## 2026-03-09

### Repo setup

- Confirmed `biowasm` is already forked under `optiarc/biowasm`.
- Repointed the Aioli submodule from upstream to `git@github.com:optiarc/aioli.git` in [.gitmodules](/home/lars/git/biowasm/.gitmodules#L1).
- Initialized the `tools/aioli/src` submodule from the fork.

### Planning

- Added [porting.md](/home/lars/git/biowasm/porting.md) describing the staged OPFS migration plan.
- Decided to keep `minimap2` unmodified for now and start with Aioli/runtime changes.

### Implemented

- Added initial OPFS utility methods to Aioli in [tools/aioli/src/src/worker.js](/home/lars/git/biowasm/tools/aioli/src/src/worker.js#L269):
  - `opfsMkdir`
  - `opfsWrite`
  - `opfsRead`
  - `opfsList`
  - `opfsDelete`
  - `copyToOpfs`
  - `copyFromOpfs`

### Tested

- Added a focused browser test in [tools/aioli/src/tests/test_opfs.cy.js](/home/lars/git/biowasm/tools/aioli/src/tests/test_opfs.cy.js#L1).
- Verified:
  - OPFS write/read/list/delete
  - persistence across a second `Aioli` instance
  - copying between OPFS and the current virtual filesystem
- `npm run build` succeeded in `tools/aioli/src`.
- `npx cypress run --spec tests/test_opfs.cy.js` passed.

### Current status

- First OPFS step is complete and browser-tested.
- OPFS exists as an explicit Aioli persistence API.
- Tool-visible OPFS paths such as `/opfs/...` are not wired yet.

### Next step completed

- Extended Aioli command execution so `exec()` can persist declared output files to OPFS after the tool finishes.
- Added support in [tools/aioli/src/src/worker.js](/home/lars/git/biowasm/tools/aioli/src/src/worker.js#L155) for:
  - `exec(command, args, { persist: ... })`
  - persistence entries in the form `{ from: "<fs-path>", to: "<opfs-path>" }`
- Added a browser test using a real command:
  - `samtools fastq ... -o toy.fastq`
  - followed by automatic persistence of `toy.fastq` into OPFS
- Verified with:
  - `npm run build`
  - `npx cypress run --spec tests/test_opfs.cy.js`

### Updated status

- Aioli now supports two OPFS usage levels:
  - explicit helper methods (`opfsWrite`, `opfsRead`, etc.)
  - command-integrated persistence via `exec(..., { persist })`
- Tools still do not see `/opfs/...` as a native writable filesystem path yet.

### Next step completed

- Added a stable staged persistent-workspace path in Aioli config:
  - [tools/aioli/src/src/main.js](/home/lars/git/biowasm/tools/aioli/src/src/main.js#L20)
  - current path: `/shared/opfs`
- Added explicit staging helpers in [tools/aioli/src/src/worker.js](/home/lars/git/biowasm/tools/aioli/src/src/worker.js#L274):
  - `opfsStage(opfsPath, fsPath?)`
  - `opfsFlush(fsPath, opfsPath?)`
- This allows tools to work with normal in-process filesystem paths under `/shared/opfs/...`, then synchronize those files to browser OPFS.

### Tested

- Extended [tools/aioli/src/tests/test_opfs.cy.js](/home/lars/git/biowasm/tools/aioli/src/tests/test_opfs.cy.js#L1) to verify:
  - a file staged from OPFS can be read by a command from `/shared/opfs/...`
  - a command can write an output file to `/shared/opfs/...`
  - that staged output can be flushed back to browser OPFS
- Verified with:
  - `npm run build`
  - `npx cypress run --spec tests/test_opfs.cy.js`

### Updated status

- Aioli now supports three OPFS-related usage levels:
  - explicit OPFS helper methods
  - `exec(..., { persist })` for direct post-command persistence
  - a staged tool-visible workspace at `/shared/opfs/...`
- This is still a staged sync model, not a native OPFS mount inside the wasm filesystem.

### Next step completed

- Added a real `minimap2` browser test for the staged OPFS workflow:
  - [tools/aioli/src/tests/test_minimap2_opfs.cy.js](/home/lars/git/biowasm/tools/aioli/src/tests/test_minimap2_opfs.cy.js#L1)
- The test:
  - initializes `minimap2/2.22` from the biowasm CDN
  - runs `minimap2` against the bundled sample FASTA files
  - writes the SAM output to `/shared/opfs/results/aln.sam`
  - flushes that file into browser OPFS at `/results/aln.sam`
  - verifies the persisted SAM content

### Tested

- Verified with:
  - `npx cypress run --spec tests/test_minimap2_opfs.cy.js`
- Result:
  - 1 passing test

### Updated status

- The staged OPFS model now works with a real `minimap2` execution path, not just helper methods or simpler tools.

### Manual verification path

- Added a small browser example for hands-on verification:
  - [tools/aioli/src/src/examples/minimap2-opfs.html](/home/lars/git/biowasm/tools/aioli/src/src/examples/minimap2-opfs.html#L1)
  - [tools/aioli/src/src/examples/minimap2-opfs.js](/home/lars/git/biowasm/tools/aioli/src/src/examples/minimap2-opfs.js#L1)
- It:
  - initializes `minimap2/2.22`
  - writes output to `/shared/opfs/results/aln.sam`
  - flushes the result to browser OPFS at `/results/aln.sam`
  - shows `stderr` and a persisted SAM preview in the page
- The example is linked from [tools/aioli/src/index.html](/home/lars/git/biowasm/tools/aioli/src/index.html#L1).
