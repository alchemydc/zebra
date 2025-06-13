# Project Brief: Project Zebra Stampede

### Introduction / Problem Statement

The core purpose of this project is to analyze, instrument, and improve the performance of the Zebra Zcash client. While Zebra historically demonstrated superior mainnet sync speeds compared to the legacy Zcashd client, its performance has notably degraded since the network's "Sandblasting" attack. The current perception is that Zebra is now slower than its predecessor, creating a critical need to identify and remediate the underlying bottlenecks to restore its high-performance status.

### Vision & Goals

* **Vision:** To restore Zebra to its position as the premier, high-performance Zcash client, trusted by the ecosystem for its speed and reliability. This will be supported by a robust, observable telemetry system that provides clear insights into performance and guides future optimizations.
* **Primary Goals:**
    1.  Develop a comprehensive performance baseline for the current Zebra client to enable precise measurement of future improvements.
    2.  Enhance Zebra's internal telemetry and visibility features to provide actionable insights into performance bottlenecks for developers.
    3.  Triage all [existing "I-slow" issues](performance_related_issues.md) in the repository to identify and prioritize work that aligns with this performance improvement initiative.
    4.  Remediate the most critical performance bottlenecks in the Zebra codebase, demonstrating measurable and consistent improvements against the established baseline.
    5.  Empower production users of Zebra to instrument, monitor, visualize, and alert on performance and potential issues within their own environments.

### Target Audience / Users

The primary users for this performance improvement project are the developers and teams who build on, maintain, or operate Zebra nodes in production environments. This includes:

* Developers within the Zcash Foundation, Electric Coin Company (ECC), Shielded Labs, QEDIT, and Zingo Labs.
* The broader Zcash ecosystem of developers and researchers who rely on Zebra for their applications and services.
* Production node operators who require high performance, stability, and robust monitoring capabilities, such as operators of:
    * Centralized and Decentralized Exchanges (DEXs)
    * Wallet services and light client providers
    * Block explorers
    * Other Zcash-powered infrastructure

### Key Features / Scope (High-Level Ideas for MVP)

* **Granular Performance Baselining:**
    * Establish and document a clear performance baseline for Zebra's initial sync time on mainnet.
    * Analyze and baseline ongoing performance under various network conditions, specifically including models that replicate the high-volume, low-value transaction environment observed during the "sandblasting attack" to understand its impact.
    * Characterize Zebra's performance and resilience during other scenarios, such as mempool saturation, network peering fluctuations, and simulated denial-of-service events.
* **Condition-Aware Telemetry & Instrumentation:**
    * Investigate all metrics currently emitted via Zebra's `--metrics` capability to determine their suitability for this project's goals.
    * Improve existing or expose new metrics to provide clear, actionable data that correlates performance with the specific network conditions mentioned above.
* **Issue Triage:**
    * Conduct a comprehensive review of all GitHub issues tagged with "I-slow" to synthesize them into the overall performance improvement plan.
* **Targeted Remediation:**
    * Identify and implement code changes to fix the highest-impact performance bottlenecks identified during both initial sync and conditional testing.

### Post MVP Features / Scope and Ideas

* **NU7 & ZSA Performance Analysis:** Proactively investigate and model the potential performance impact of Zcash Shielded Assets (ZSAs) ahead of the NU7 network upgrade.
* **Z3 Stack Compatibility:** Ensure any performance testing harness or tooling developed during the MVP is designed to be forward-compatible with the future "Z3" stack (Zebra, Zaino, Zallet).
* **Automated Performance Regression Suite:** Develop a fully automated performance regression testing suite that runs on a regular cadence (e.g., nightly) to catch performance degradations early.
* **Public Performance Dashboard:** Create a public-facing dashboard to transparently display Zebra's key performance metrics to the Zcash community.
* **Experimental Optimizations:** Investigate more speculative or long-term architectural changes for performance that are out of scope for the initial remediation effort.

### Known Technical Constraints or Preferences

* **Constraints:**
    * **Deadline:** All identified performance remediations must be tested and deployed before the end of Q4 2025.
    * **Compliance:** No specific compliance standards are required for this project.
* **Technical Preferences:**
    * **Languages:** The Zebra codebase is written in Rust, so all changes must be in Rust.
    * **Tooling:** The project should continue to use Prometheus for metrics collection and Grafana for visualization.
* **Risks:**
    * **Reproducibility:** Initial sync performance issues may be difficult to reproduce consistently, as some CI environments have demonstrated fast sync times. This may complicate baselining and remediation validation.
* **Key Dependencies & Collaboration:**
    * **ECC Collaboration:** The ECC team implemented performance improvements in `zcashd` during the "sandblasting attack" that were not mirrored in Zebra. A high-priority initial action is to collaborate with ECC engineers to understand these changes and formulate a plan to implement equivalent improvements in Zebra.

