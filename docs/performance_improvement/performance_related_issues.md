# Performance-Related Issues in ZcashFoundation/zebra

_Last updated: 2025-06-13_

**Instructions for updating this file:**  
Use the following prompt to regenerate the summaries (and be sure to invoke the GitHub MCP tool as described):

> "Use the GitHub MCP server's `list_issues` tool with parameters: owner='ZcashFoundation', repo='zebra', labels=['I-slow'], state='open' to fetch all open issues with the 'I-slow' label. Then, create a markdown file 'performance_related_issues.md' that provides a brief summary of each issue. Add a last updated date and instructions for updating the file (the prompt used) to the top of the file."

---

This document summarizes all open issues labeled "I-slow", "A-diagnostics", or "I-unbounded-growth" in the [ZcashFoundation/zebra](https://github.com/ZcashFoundation/zebra) repository.

---

## [#9527: CI applies the same tests multiple times](https://github.com/ZcashFoundation/zebra/issues/9527)
CI runs nearly identical test sets three times per PR, significantly increasing the time to see results on main. Reducing redundancy would streamline the release process.

---

## [#9482: Cache compilation results in `runtime` builds](https://github.com/ZcashFoundation/zebra/issues/9482)
CI recompiles Zebra for the `runtime` Docker target at each commit, taking about 9 minutes each time. Adding caching would reduce build times.

---

## [#9331: CI recompiles Zebra from scratch for each CI test](https://github.com/ZcashFoundation/zebra/issues/9331)
Each PR update triggers full recompilation for every test set, causing excessive build times. Improved caching of crates and compilation results is needed.

---

## [#9165: Compute the tx SIGHASH only once per tx verification](https://github.com/ZcashFoundation/zebra/issues/9165)
The transaction SIGHASH is computed twice per verification due to FFI limitations. Refactoring to compute it once would improve efficiency.

---

## [#9162: Finalize blocks in parallel with contextual validation](https://github.com/ZcashFoundation/zebra/issues/9162)
Block finalization currently blocks contextual validation, limiting sync performance. Parallelizing these operations could speed up initial sync.

---

## [#8044: security: Run services that process external data it their own tokio executors, to avoid denial of service attacks](https://github.com/ZcashFoundation/zebra/issues/8044)
Services that process external data are vulnerable to denial of service attacks and unbounded resource growth. Running these services in separate executors can limit their impact and improve overall robustness.

---

## [#7981: The number of outbound connections reported by progress bars periodically glitches](https://github.com/ZcashFoundation/zebra/issues/7981)
On Testnet, the outbound connection count in progress bars jumps from ~7 to 75 and back, likely due to how connection limits are updated. Moving the progress bar logic to the peer set could resolve this.

---

## [#7939: perf: Stop expensive cryptographic validation when deserializing shielded transactions](https://github.com/ZcashFoundation/zebra/issues/7939)
Deserialization of shielded transactions performs unnecessary cryptographic checks. Skipping these for known-valid transactions would save CPU.

---

## [#7852: Make the outbound connection rate slower when there are lots of recent outbound connections](https://github.com/ZcashFoundation/zebra/issues/7852)
The current fixed outbound connection rate can cause network stress. Dynamically adjusting the rate could prevent denial-of-service scenarios.

---

## [#7416: diagnostic: Log column family and database size on startup and shutdown](https://github.com/ZcashFoundation/zebra/issues/7416)
Users and developers want better visibility into database and memory usage. Logging column family and database sizes at startup and shutdown would help monitor resource growth.

---

## [#6714: Stop building a separate `end_of_support` integration test binary for `zebrad`](https://github.com/ZcashFoundation/zebra/issues/6714)
Building a separate test binary increases build time and disk usage. Consolidating tests would improve efficiency.

---

## [#6068: Add `startingheight` field to `getpeers` RPC](https://github.com/ZcashFoundation/zebra/issues/6068)
Adding a `startingheight` field to the `getpeers` RPC would improve mining pool compatibility and user experience by providing more accurate sync progress information.

---

## [#5709: Fix repeated block timeouts during initial sync](https://github.com/ZcashFoundation/zebra/issues/5709)
Block validation timeouts and sync resets occur repeatedly during initial sync, slowing down the process. Adjusting timeouts and pipeline capacity may help.

---

## [#5604: Send the same getblocktemplate RPC response until the template would change](https://github.com/ZcashFoundation/zebra/issues/5604)
Caching the `getblocktemplate` RPC response could improve RPC performance for miners, as the current implementation recalculates it too often.

---

## [#5297: Add metrics for chain fork work and lengths](https://github.com/ZcashFoundation/zebra/issues/5297)
ZIP editors requested metrics for chain forks and non-finalized chain lengths. Adding these metrics would improve diagnostics and monitoring of chain health.

---

## [#4672: Add support for Orchard proof batch verification](https://github.com/ZcashFoundation/zebra/issues/4672)
Batch verification for Orchard proofs is not yet used, leading to slow validation for blocks with many actions. Implementing this would improve performance.

---

## [#3153: Tracking: Improve performance](https://github.com/ZcashFoundation/zebra/issues/3153)
A tracking issue for various performance improvements, including benchmarking and optimizing CPU usage, note commitment tree updates, and database writes.

---

## [#1875: Zebra attempts new peer connections in a fixed, predictable order](https://github.com/ZcashFoundation/zebra/issues/1875)
Peer connection attempts follow a predictable order, which can be exploited and may not be optimal for performance. Randomizing the selection could help.
