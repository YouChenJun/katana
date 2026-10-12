# Interactive crawl and complete script capture plan

## Goal and coverage contract

Add an opt-in pure-headless crawl that explores reachable UI states to a user-selected click depth and saves every completed JavaScript response it observes, including chunks loaded by later interactions. “Complete” means the body returned by Chrome for a finished request is written byte for byte and verified by size and SHA-256. Cancelled, failed, oversized, and unavailable responses must be reported as incomplete, never silently treated as complete. Code that the browser never requests cannot be guaranteed; statically discoverable nested imports should be optionally fetched and marked as discovered rather than observed.

## Implementation stages

1. **Options and scope.** Keep current behavior unchanged unless interactive mode is selected. Expose interaction depth, candidate selector and exclusions, maximum states/actions/duration, click settling timeout, script output directory, script discovery depth/request budget, and static asset scope. Page navigation stays under the existing crawl scope. Browser-observed CDN scripts can be saved even if outside page scope. Continue the ordinary headless queue after UI exploration; `-interaction-duration` reserves time for it within `-crawl-duration`.
2. **State exploration.** Use depth-first traversal on one authenticated browser session. A state is URL plus a DOM fingerprint. Deduplicate discovered states globally. After finishing a branch, restore the parent via browser history when it navigated, otherwise recreate it by replaying the recorded path. Count only successful transitions toward interaction depth. Avoid restoring merely because a live page changed in the background before the first action. Adopt new tabs opened by a click, reload after installing response hooks, explore them as child states, then close them. Closed shadow roots and cross-origin frames remain coverage limits.
3. **Candidate discovery and settling.** Include native buttons/links, semantic roles, inline click handlers and observed click listeners. Re-query controls after each state change. Check visibility, coverage, and disabled status immediately before clicking, then dispatch browser mouse events without Rod's unbounded animation stability wait. Wait for navigation, DOM change, or bounded network quiet. Expose selectors and limits to users.
4. **Script capture.** Use the existing CDP response interception. For script resources, preserve decoded response bytes before ordinary output filtering, parsing, truncation or `-omit-body`. Save atomically to a temporary file, fsync/close, rename, then record URL, status, byte count, SHA-256, MIME, source and any error in a manifest. Keep multiple URL variants and revisions distinct. Treat failed CDP body retrieval or an HTTP error as incomplete. Do not compare Content-Length to decoded body size when transfer compression is used.
5. **Nested scripts.** Browser-triggered `import()`, workers and script elements are captured as they load. Add optional bounded static discovery for literal `import`, `export ... from`, `importScripts`, and `new Worker`. Resolve against the containing script URL. Fetch discovered files with the same authenticated browser context, depth/request/origin limits and provenance; mark these separately from scripts actually executed. Dynamic computed URLs remain an explicit coverage gap.
6. **Cookies.** Existing `-H 'Cookie: ...'` works in the standard and hybrid engines but is not forwarded to the pure-headless browser. Parse it separately from other headers, seed scoped cookies before the first navigation, and keep ordinary custom headers for requests. Existing `-cwu` and `-no-incognito -cdd` sessions must retain their browser cookies. Never log cookie values or write them into the script manifest.

## Verification

- Unit tests: atomic saving and manifest outcomes, concurrent deduplication, static import resolution, discovery origin checks, cookie parsing/domain/path rules.
- Local browser fixture: four nested UI levels with sibling branches, a repeated button in different states, lazy `import()` chunks, a static two-level import, a 2 MB script, a gzip-compressed script, and an authenticated route that requires a seeded cookie.
- Measure baseline `-hl` against the new mode on the same fixture: unique UI states, successful actions at each depth, unique script bodies, incomplete downloads, elapsed time, bytes transferred and browser requests. Profile time in click settling, state restoration, DOM extraction and response interception before tuning concurrency.
- After local correctness, run bounded, low-rate, unauthenticated comparisons on a small set of public Baidu and QQ subdomain pages. Record exact targets, date, flags and scope. Compare unique URLs, paths, states and JS resources; count redirects and failed responses separately. Do not present these public-site results as a guarantee of complete coverage.

## Performance decisions

- Start with one browser session and DFS to minimize replay between nearby states. Reuse loaded script bodies by content hash; do not refetch them merely to save them.
- Bound work with state/action/time/static-discovery limits and expose the limits as flags. Complete browser-observed scripts can be large, so disk capacity remains a user consideration.
- Add browser concurrency only after measuring a bottleneck: it complicates cookie state, reproducibility and response provenance.

## Current coverage limits

The implemented traversal handles controls in the main document and new tabs with navigable in-scope URLs. A control inside a frame or closed shadow root, canvas-only controls, and blank popups populated only by script are not yet traversed as child states. No crawler can infer every computed import URL without executing the code path that builds it. The manifest distinguishes fully captured responses from errors; a request aborted before Chrome exposes its response body may not appear in the manifest. A future extension can attach to targets before their first navigation and add frame/shadow-root locators with corresponding restoration paths.

## 2026-09-29 verification record

The deterministic local comparison used the original `09c83e9` binary and the modified binary against the same four-level fixture with a 30-second cap. The original completed in 13 seconds and emitted 5 unique endpoints, reaching only `chunk1.js`. The modified run completed in 18 seconds, explored 9 states with 9 successful clicks, emitted 8 unique endpoints, and added `chunk2.js`, `chunk3.js`, and `chunk4.js`. It saved 9 complete scripts, including two unexecuted nested Worker references, with no unresolved script URL. The browser E2E test separately checks exact 2 MB and gzip-decoded bodies, a cookie-protected root, and an in-scope popup with a lazy chunk; it reaches 11 states with 13 successful clicks. The entire Go test suite passed with `CGO_ENABLED=0 go test ./... -count=1` after the authentication timeout fix; focused tests cover the final script parser and popup changes.

Public runs used one input at a time, `-d 2 -ct 45s -c 1 -p 1 -rl 1`, no credentials, and a 12-action/30-state interaction cap. "Paths" below are unique `(host, pathname)` pairs, while "directories" are their path prefixes. The public pages changed between runs, and session identifiers appear in some paths, so new-only figures show observed differences rather than causally proven interaction discoveries.

| Target | Original URLs | Modified URLs | Only modified URLs | Only modified paths | Only modified directories | Complete JS | Unresolved JS URLs |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| `www.baidu.com` | 128 | 75 | 1 | 0 | 0 | 33 | 0 |
| `m.baidu.com` | 455 | 448 | 320 | 90 | 31 | 142 | 0 |
| `news.qq.com` | 209 | 261 | 76 | 7 | 1 | 29 | 0 |
| `mail.qq.com` | 53 | 91 | 64 | 2 | 0 | 67 | 6 |

The modified runs used an 8- or 15-second UI budget and a 5- or 10-second script finalization reserve within the 45-second cap. The finalization phase resumes ordinary crawling if it finishes early. The `news.qq.com` row is from a final-version smoke run after popup support; its ordinary queue stopped after ten consecutive failed actions with 361 actions still queued, so the row reflects a bounded partial crawl. On `mail.qq.com`, six unresolved URLs were time-varying login JSONP polls; Chrome timed out on their original responses and returned no resource stream on retry. The manifest retains those failures rather than claiming a complete body.
