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
  - writes output to `/opfs/results/aln.sam`
  - flushes the result to browser OPFS at `/results/aln.sam`
  - shows `stderr` and a persisted SAM preview in the page
- The example is linked from [tools/aioli/src/index.html](/home/lars/git/biowasm/tools/aioli/src/index.html#L1).

### Next step completed

- Switched Aioli to expose `/opfs` as the public caller-facing path while keeping `/shared/opfs` as the internal staged backend for now.
- Added path translation in [tools/aioli/src/src/worker.js](/home/lars/git/biowasm/tools/aioli/src/src/worker.js#L726) so:
  - command arguments targeting `/opfs/...` are rewritten to the staged backend
  - file utility methods like `mkdir`, `pwd`, `read`, `write`, `cat`, `ls`, and `download` can accept `/opfs/...`
  - `opfsStage()` returns public `/opfs/...` paths
- Updated tests and the manual `minimap2` example to use `/opfs/...` instead of `/shared/opfs/...`.

### Tested

- Verified with:
  - `npm run build`
  - `npx cypress run --spec tests/test_opfs.cy.js`
  - `npx cypress run --spec tests/test_minimap2_opfs.cy.js`

### Updated status

- Callers can now target `/opfs/...`, which matches the intended future direct-mount API shape.
- Internally this is still a staged sync model backed by `/shared/opfs/...`.

### Next step completed

- Isolated the current staged implementation behind an explicit OPFS backend abstraction in Aioli.
- Added `opfsBackend: "staged"` in [tools/aioli/src/src/main.js](/home/lars/git/biowasm/tools/aioli/src/src/main.js#L17).
- Centralized backend path handling in [tools/aioli/src/src/worker.js](/home/lars/git/biowasm/tools/aioli/src/src/worker.js#L726):
  - `_opfsBackendRoot()`
  - `_resolveFsPath()`
  - `_publicFsPath()`
- Updated the worker to use that abstraction for:
  - command arguments
  - `mkdir`, `cd`, `pwd`, `read`, `write`
  - OPFS stage/flush helpers
  - backend-root initialization

### Tested

- Extended [tools/aioli/src/tests/test_opfs.cy.js](/home/lars/git/biowasm/tools/aioli/src/tests/test_opfs.cy.js#L1) so it now also verifies:
  - `opfsStage()` returns the public `/opfs/...` path
  - `pwd()` reports the public `/opfs/...` path after `cd("/opfs/...")`
- Verified with:
  - `npm run build`
  - `npx cypress run --spec tests/test_opfs.cy.js`
  - `npx cypress run --spec tests/test_minimap2_opfs.cy.js`

### Updated status

- The staged implementation is now more cleanly encapsulated.
- Replacing it with a future direct OPFS mount should require fewer caller-facing changes because `/opfs/...` and the backend boundary are now explicit.

### Next step completed

- Split the staged implementation further into backend-specific methods in [tools/aioli/src/src/worker.js](/home/lars/git/biowasm/tools/aioli/src/src/worker.js#L765):
  - `_opfsBackendInitFS(FS)`
  - `_opfsBackendStageFromOpfs(opfsPath, fsPath)`
  - `_opfsBackendFlushToOpfs(fsPath, opfsPath)`
- Added `opfsBackend: "staged"` to the default config in [tools/aioli/src/src/main.js](/home/lars/git/biowasm/tools/aioli/src/src/main.js#L17).
- This moves more of the current behavior behind a backend boundary so a future direct-mount backend can be added alongside the staged one.

### Tested

- Updated [tools/aioli/src/tests/test_opfs.cy.js](/home/lars/git/biowasm/tools/aioli/src/tests/test_opfs.cy.js#L1) to exercise the explicit `opfsBackend: "staged"` config path.
- Verified with:
  - `npm run build`
  - `npx cypress run --spec tests/test_opfs.cy.js`
  - `npx cypress run --spec tests/test_minimap2_opfs.cy.js`

### Updated status

- The next major implementation step is now clearer: add a second backend mode for direct OPFS mounting, then point the same `/opfs/...` public API at that backend instead of the staged one.

### Next step completed

- Added an explicit `direct` backend scaffold in Aioli.
- The code now distinguishes:
  - `opfsBackend: "staged"`: current working implementation
  - `opfsBackend: "direct"`: reserved for the future true direct-mount implementation
- The `direct` mode currently fails fast with a clear error instead of silently falling back to staged behavior.

### Tested

- Extended [tools/aioli/src/tests/test_opfs.cy.js](/home/lars/git/biowasm/tools/aioli/src/tests/test_opfs.cy.js#L1) with a test that verifies `opfsBackend: "direct"` fails with a clear “not implemented” message.
- Verified with:
  - `npm run build`
  - `npx cypress run --spec tests/test_opfs.cy.js`
  - `npx cypress run --spec tests/test_minimap2_opfs.cy.js`

### Updated status

- The codebase now has a concrete place to implement the future direct OPFS backend without reworking the caller-facing `/opfs/...` contract again.

### Next step in progress

- Added an experimental direct `/opfs` backend in [tools/aioli/src/src/worker.js](/home/lars/git/biowasm/tools/aioli/src/src/worker.js#L1) that mounts a custom Emscripten filesystem in the worker and backs explicit `/opfs/...` files with OPFS sync access handles.
- The direct backend now:
  - prepares explicit output paths such as `-o /opfs/results/aln.sam` before command execution
  - lets tools read and write those prepared files through normal filesystem calls
  - keeps `/results/...` persisted in browser OPFS without a post-run `opfsFlush()` copy
- Updated the manual minimap2 verification page to use `opfsBackend: "direct"`:
  - [tools/aioli/src/src/examples/minimap2-opfs.html](/home/lars/git/biowasm/tools/aioli/src/src/examples/minimap2-opfs.html#L1)
  - [tools/aioli/src/src/examples/minimap2-opfs.js](/home/lars/git/biowasm/tools/aioli/src/src/examples/minimap2-opfs.js#L1)
- Updated the OPFS Cypress specs to target the direct backend for explicit output files:
  - [tools/aioli/src/tests/test_opfs.cy.js](/home/lars/git/biowasm/tools/aioli/src/tests/test_opfs.cy.js#L1)
  - [tools/aioli/src/tests/test_minimap2_opfs.cy.js](/home/lars/git/biowasm/tools/aioli/src/tests/test_minimap2_opfs.cy.js#L1)

### Tested

- Verified with:
  - `npm run build`
- Browser verification is currently blocked on this machine because Cypress cannot start `Xvfb`; `/tmp/.X11-unix` is owned by the local user instead of `root`, so the new direct-backend specs could not be executed here yet.

### Manual verification completed

- Verified the direct minimap2 flow interactively in the browser via:
  - [tools/aioli/src/src/examples/minimap2-opfs.html](/home/lars/git/biowasm/tools/aioli/src/src/examples/minimap2-opfs.html#L1)
- Result:
  - the page completed successfully
  - `minimap2-simd` wrote to `/opfs/results/aln.sam`
  - the file was immediately available in browser OPFS as `/results/aln.sam`
  - the example no longer required an explicit `opfsFlush()` step
- Observed output included:
  - minimap2 `stderr` timing and command logs
  - persisted SAM content read back from OPFS

### Updated status

- The direct backend is now manually validated for the core `minimap2 -o /opfs/...` workflow.
- This is still an experimental Aioli-side direct filesystem implementation, but it proves the intended `minimap2` direct-OPFS path works end-to-end.

### Next step completed

- Added a dedicated `samtools` OPFS support document:
  - [samtools-opfs-support.md](/home/lars/git/biowasm/samtools-opfs-support.md#L1)
- Added a dedicated `samtools` OPFS Cypress spec in Aioli:
  - [tools/aioli/src/tests/test_samtools_opfs.cy.js](/home/lars/git/biowasm/tools/aioli/src/tests/test_samtools_opfs.cy.js#L1)
- The first version of that spec hit a browser worker `importScripts()` loading issue under Cypress 15 when Aioli was initialized directly inside the spec.
- Switched the `samtools` OPFS checks to a page-harness model:
  - [tools/aioli/src/src/examples/samtools-opfs-test.html](/home/lars/git/biowasm/tools/aioli/src/src/examples/samtools-opfs-test.html#L1)
  - [tools/aioli/src/src/examples/samtools-opfs-test.js](/home/lars/git/biowasm/tools/aioli/src/src/examples/samtools-opfs-test.js#L1)
- Updated the Cypress config in [tools/aioli/src/cypress.config.js](/home/lars/git/biowasm/tools/aioli/src/cypress.config.js#L1) and upgraded Aioli's Cypress dependency to the 15.x line so browser tests can run on this host with:
  - `DISPLAY=:99`
  - `--browser chromium`

### Tested

- Verified with:
  - `npm run build`
  - `DISPLAY=:99 npx cypress verify`
  - `DISPLAY=:99 npx cypress run --browser chromium --spec tests/test_samtools_opfs.cy.js`
- Result:
  - `1 passing`
  - `0 failing`
  - `3 pending`

### Updated status

- The explicit-output `samtools` OPFS workflows are now browser-tested through the harness page:
  - `samtools view -o /opfs/...`
  - `samtools fastq -o /opfs/...`
  - OPFS input to explicit OPFS output
- `sort`, `index`, and `faidx` remain intentionally pending because they exercise the incomplete implicit-file and sidecar behaviors in the current direct backend.

### Next step completed

- Converted the backend split into an explicit in-code backend contract in [tools/aioli/src/src/worker.js](/home/lars/git/biowasm/tools/aioli/src/src/worker.js#L770).
- The contract now defines:
  - `root`
  - `initFS(FS)`
  - `stageFromOpfs(opfsPath, fsPath)`
  - `flushToOpfs(fsPath, opfsPath)`
- The staged backend implements that contract today.
- The direct backend now exists as a structured placeholder behind the same contract.

### Tested

- Verified with:
  - `npm run build`
  - `npx cypress run --spec tests/test_opfs.cy.js`
  - `npx cypress run --spec tests/test_minimap2_opfs.cy.js`

### Updated status

- The next direct-OPFS implementation step now has a precise target interface inside the codebase.
- That should make the eventual runtime/toolchain migration less invasive because the public `/opfs/...` API and the internal backend contract are both already fixed.

### Next step completed

- Tightened the `direct` backend placeholder so it now points at the expected future integration hook:
  - `initFS(FS, module)` can use `module.mountOpfs(...)` once the runtime/toolchain supports it
- Updated the error path so `direct` now fails with an explicit message about missing module/runtime OPFS mount support, instead of a generic “not implemented” message.

### Tested

- Updated [tools/aioli/src/tests/test_opfs.cy.js](/home/lars/git/biowasm/tools/aioli/src/tests/test_opfs.cy.js#L1) so the direct-backend failure test now checks for the runtime-support message.
- Verified with:
  - `npm run build`
  - `npx cypress run --spec tests/test_opfs.cy.js`
  - `npx cypress run --spec tests/test_minimap2_opfs.cy.js`

### Updated status

- The code now identifies the exact future hook where a direct backend should attach to a module/runtime implementation.

### Next step completed

- Defined the required biowasm module/runtime hook for the future direct backend in [module-opfs-hook.md](/home/lars/git/biowasm/module-opfs-hook.md#L1).
- The hook contract is:
  - `await module.mountOpfs({ root: "/opfs", FS: module.FS })`
- The document now defines:
  - what the hook must do
  - what it must not do
  - the split between Aioli responsibilities and biowasm module responsibilities
  - acceptance criteria for replacing the staged copy path
- Added a pointer comment in [bin/shared.sh](/home/lars/git/biowasm/bin/shared.sh#L1) so the shared build layer now references this future runtime requirement explicitly.

### Updated status

- The future direct-backend boundary is now defined on both sides:
  - Aioli side: `opfsBackend: "direct"` and the backend contract
  - biowasm side: the required `module.mountOpfs(...)` hook contract

### Next step completed

- Updated the shared biowasm glue-finalization step in [bin/compile.sh](/home/lars/git/biowasm/bin/compile.sh#L53) so generated module wrappers now expose a default `mountOpfs()` stub.
- The stub currently throws a clear error telling the caller that the module must be rebuilt with direct OPFS support.
- This creates a consistent generated-module boundary for the future direct backend, instead of leaving the hook entirely hypothetical.

### Tested

- Verified the updated build script parses cleanly with:
  - `bash -n bin/compile.sh`

### Updated status

- The future `mountOpfs()` hook now exists at three levels:
  - Aioli direct backend placeholder
  - biowasm hook design document
  - generated-module wrapper stub in the shared build pipeline

### Next step completed

- Added explicit runtime-side capability inspection in Aioli via `opfsSupport(...)`.
- This reports whether the current module appears to have real direct OPFS mount support or only a stub/unsupported path.
- Aioli's direct backend detection now uses that capability distinction instead of checking only for the existence of a function name.

### Tested

- Extended [tools/aioli/src/tests/test_opfs.cy.js](/home/lars/git/biowasm/tools/aioli/src/tests/test_opfs.cy.js#L1) with a capability-inspection test for the current staged modules.
- Verified with:
  - `npm run build`
  - `npx cypress run --spec tests/test_opfs.cy.js`
  - `npx cypress run --spec tests/test_minimap2_opfs.cy.js`

### Updated status

- The future direct-backend rollout can now be gated on an explicit runtime capability signal instead of implicit error behavior.

### Next step completed

- Added generated module capability metadata for direct OPFS support in [bin/compile.sh](/home/lars/git/biowasm/bin/compile.sh#L53):
  - `module.biowasmCapabilities.mountOpfs`
- Updated Aioli's `opfsSupport(...)` to prefer that capability metadata when present, while keeping a fallback for older modules that only expose the function/stub shape.

### Tested

- Verified the shared build script still parses cleanly with:
  - `bash -n bin/compile.sh`

### Updated status

- The future direct-backend handshake is now cleaner:
  - generated module wrapper advertises `mountOpfs` capability
  - Aioli can consume that capability explicitly

### Next step completed

- Extended generated module capability metadata with `mountOpfsSource` in [bin/compile.sh](/home/lars/git/biowasm/bin/compile.sh#L53).
- Updated Aioli's `opfsSupport(...)` to report a `source` field such as:
  - `metadata`
  - `stub`
  - `missing`
  - `function`
  - `uninitialized`
- This gives the future direct-backend rollout a clearer diagnostic path when modules begin transitioning from stubbed to real support.

### Tested

- Verified with:
  - `bash -n bin/compile.sh`
  - `npm run build`

### Updated status

- The direct-backend capability handshake now includes both:
  - whether `mountOpfs` is supported
  - where that support signal came from

### Next step completed

- Routed the `direct` backend through Aioli's explicit capability check instead of ad hoc function checks.
- Added `_requireDirectOpfsSupport(...)` in [tools/aioli/src/src/worker.js](/home/lars/git/biowasm/tools/aioli/src/src/worker.js#L320) so direct-backend initialization now fails with the normalized capability reason.

### Tested

- Verified with:
  - `npm run build`
  - `npx cypress run --spec tests/test_opfs.cy.js`
  - `npx cypress run --spec tests/test_minimap2_opfs.cy.js`

### Updated status

- The direct backend is now gated through a single capability path, which should make the future transition from “unsupported” to “supported” simpler and less error-prone.
