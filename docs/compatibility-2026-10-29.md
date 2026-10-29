# Compatibility verification — 2026-10-29

This check starts from upstream commit `65bea9f85cbf046b83ad199568a474ed0a74f003` (2.2.0). It verifies the built MCP server over stdio, rather than substituting mocks for its public tools.

## Fix

The compile client always sent `rootResourcePath: "main.tex"`. That bypasses Overleaf's saved main-document selection. The client now omits that field unless explicitly requested, allowing Overleaf's `CompileManager` to resolve the selected root document. This applies to compile, log and PDF tools.

Two REST regression tests cover a selected root with a different name and an explicitly supplied root. The CE live compile scenario also renames the root to `paper.tex` and moves it into a subfolder before compiling.

## Local checks

- `npm run typecheck`: passed.
- `npm test`: 315 tests passed across 39 files, including a real headless-browser login test.
- `npm run build`: passed.
- `node scripts/smoke-stdio.mjs overleaf_list_hosts`: MCP initialization and discovery of all 22 tools passed; configured host discovery succeeded.
- `git diff --check` and `bash -n test/live/run-matrix.sh`: passed.

The new default-root regression test failed on the original implementation (`main.tex` instead of the configured root), then passed after the fix.

## Production overleaf.com

The user completed browser login. A new, dedicated scratch project was created for this run. Its root document was renamed and moved to `build-source/paper.tex` before the live suite started. The project uses **classic ShareJS text OT**, as confirmed by a separate `joinDoc` connection.

`LIVE_HOST=overleaf.com LIVE_PROJECT=<scratch-project-name> npm run test:live` passed all 10 applicable scenarios in 93.53 seconds. The bootstrap test and history-OT-only write-gate test were skipped because this was a configured classic-OT host.

Verified: project discovery; text reads and edits, including CJK; create/rename/move/delete; binary upload/download; non-BMP normalization; overlapping edits from independent clients; overwrite protection; comment create/reply/resolve lifecycle; compilation; compile log; and PDF download. The concurrency tests check that clients converge without OT rejection or disconnection. The second client uses this repository's engine, not a manually operated browser.

An additional real-server negative control used the previous request (`rootResourcePath: "main.tex"`) on the same scratch project: `failure`, no PDF. The fixed request returned `success`, its log named `paper.tex`, and the downloaded PDF had a valid `%PDF-` header (16,842 bytes).

## Community Edition 6.3.2

The [official image catalogue](https://hub.docker.com/r/sharelatex/sharelatex/tags) listed `latest` as 6.3.2, published on 2026-10-23. The tested image was `sharelatex/sharelatex:6.3.2`, digest `sha256:ada39211c54a1e748065834ec44d3a49094cc475c28d8b803a251901592a94b2`.

`DOCKER_DEFAULT_PLATFORM=linux/amd64 bash test/live/run-matrix.sh 6.3.2` passed 21 applicable tests in 69.37 seconds for the main suite. The bootstrap test and classic project's history-OT-only test were skipped in that phase; bootstrap had already run separately to create the projects. Both classic and history-OT projects passed concurrent editing, renamed/nested-root compilation, log and PDF retrieval. The history-OT suite also verified the default read-only gate. CE correctly refused unsupported comment operations without changing the document.

The matrix checked that no ports were published and that the sandbox network was internal. The server's own logs recorded **zero OT errors**. Only resources created by the run were removed afterwards.

## Verification boundaries

- A migrated **history-OT project on overleaf.com was not available** in this run. Its writes remain opt-in through `OVERLEAF_HISTORY_OT_WRITES=1`, and attaching comments on that format remains unsupported. Passing classic-OT production tests does not establish compatibility for those operations.
- Server Pro and older CE versions were not independently re-tested in this run. Existing unit coverage and the rest of the version matrix remain available.
- Online results establish behavior on the date above; overleaf.com does not expose a pinned release for reproducing the exact deployment later.
- These are unreleased source changes. Build this checkout with `npm ci && npm run build` and run `node dist/cli.js` (or the plugin launcher, which prefers the local build). An unbuilt plugin checkout still falls back to the published npm version and does not include this fix.
