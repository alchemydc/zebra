# Project Zebra Stampede Product Requirements Document (PRD)

## Goals and Background Context

### Goals
* Develop a comprehensive performance baseline for the current Zebra client.
* Enhance Zebra's internal telemetry and visibility features for developers.
* Triage all existing "I-slow" issues to prioritize work.
* Remediate the most critical performance bottlenecks in the codebase.
* Empower production users to instrument, monitor, visualize, and alert on performance.

### Background Context
This project aims to analyze, instrument, and improve the performance of the Zebra Zcash client. With the upcoming deprecation of `zcashd`, Zebra is the cornerstone of the new Z3 stack, making its performance critical for future ecosystem adoption. Historically, Zebra was faster at syncing the mainnet than `zcashd`, but its performance has degraded since the "Sandblasting" network attack. This PRD outlines the requirements to identify and remediate these performance bottlenecks, restoring Zebra's status as a high-performance client.

## Requirements

### Success Metric
* The primary performance target is for Zebra's initial sync time to be **equal to or faster than** the current `zcashd` client on a comparable mainnet snapshot.

### Functional
* **FR1**: The system must expose detailed performance and diagnostic metrics via its Prometheus-compatible listener (`--metrics` flag).
* **FR2**: The system must emit structured logs (e.g., via `systemd-journal`) containing sufficient detail to diagnose performance bottlenecks and trace execution-flow timing.
* **FR3**: The Zebra codebase must be structured to allow developers to easily add or modify metrics and structured log points with minimal effort.
* **FR4**: The system's combined observability outputs (metrics, logs, and traces) from the above must be sufficient to establish a comprehensive performance baseline for both initial sync and ongoing operations.
* **FR5**: The observability data must contain labels and contextual information sufficient to correlate Zebra's performance with specific network conditions (e.g., high transaction volume, mempool saturation).
* **FR6**: Any code remediations must result in measurable improvements against the performance baseline, as verified using the enhanced observability outputs.

### Non-Functional

#### Project Constraints & Deliverables
* **NFR1**: All performance remediation work scoped for this project must be tested and deployed by the end of Q4 2025.
* **NFR2**: The project should deliver a default, version-controlled Grafana dashboard configuration for visualizing key performance metrics. (Stretch Goal)
* **NFR3**: The project must produce documentation guiding operators on how to configure and use the enhanced observability features.
* **NFR4**: The remediation plan for Zebra must be informed by a technical analysis of performance improvements previously implemented in `zcashd`.
* **NFR5**: The project's scope includes the triage and analysis of all ["I-slow" labeled issues](performance_related_issues.md) in the Zebra GitHub repository to inform the remediation backlog.

#### Technical Requirements
* **NFR6**: All new code and modifications must be written in stable Rust, adhering to existing project conventions.
* **NFR7**: The system's telemetry must be compatible with Prometheus for collection and Grafana for visualization.
* **NFR8**: The project must create the necessary DevOps tooling and infrastructure, separate from the existing correctness-focused CI pipeline, to enable long-term observation and performance instrumentation of Zebra.

## Technical Assumptions
* **Repository Structure:** Zebra is a **Monorepo**. The root of the repository contains a `Cargo.toml` file with a `[workspace]` definition that includes the main `zebrad` application and all its library crates.
* **Service Architecture:** Zebra is best described as a **Modular Monolith**. It runs as a single `zebrad` process but is internally composed of several distinct, reusable library crates.
* **Testing requirements:** The project has a multi-layered testing strategy including Rust-based unit/integration tests run via `cargo test`, Python-based E2E/RPC tests, and automated execution in the CI pipeline.
* **Additional Technical Assumptions and Requests:**
    * **Language:** Rust
    * **Instrumentation:** Prometheus
    * **Visualization:** Grafana

## Epics

* **Epic 1: Foundational Observability & Analysis:** Establish the necessary tooling, metrics, and logs to accurately measure Zebra's performance and complete the initial analysis of known issues.
* **Epic 2: Performance Baselining & Bottleneck Identification:** Utilize the new observability tooling to conduct comprehensive performance tests under various conditions and pinpoint specific root causes for slowdowns.
* **Epic 3: Targeted Remediation and Validation:** Implement and verify code changes that directly address the identified bottlenecks to meet the project's performance goals.

---

## Epic 1: Foundational Observability & Analysis
This epic is about building the foundational infrastructure for the entire project. The core deliverables will be the DevOps tooling for long-term performance observation and the initial enhancements to Zebra's metric and logging output. The success of this epic is defined by the team having a reliable, repeatable way to deploy a Zebra node and observe its performance under specific, manually configured, conditions.

### Story 1.1 DevOps Observation Environment Setup
As a **Performance Engineer**, I want **a dedicated environment for running and observing Zebra**, so that **I can collect consistent performance data over time.**
#### Acceptance Criteria
1. The environment includes a mechanism to deploy a specific version of `zebrad` and configure it to target a specific network (e.g., Mainnet or Testnet).
2. The environment includes a Prometheus-compatible metrics collection service (e.g., GCP Managed Service for Prometheus) and a Grafana instance (e.g., Grafana Cloud or self-hosted), pre-configured to scrape and visualize metrics from the deployed `zebrad` instance.
3. The environment must provide a mechanism for collecting, indexing, and visualizing structured logs (e.g., from `journald`), such as GCP's Log Explorer or a similar log analysis tool.
4. The setup is documented and can be reliably reproduced by other engineers on the team.
5. The environment must be separate from both the existing CI infrastructure and production infrastructure.

### Story 1.2 `zcashd` Performance Analysis
As a **Zebra Developer**, I want **a technical summary of `zcashd`'s sandblasting-related performance improvements**, so that **I can inform Zebra's instrumentation and remediation plan.**
#### Acceptance Criteria
1. The summary identifies the specific code changes, architectural patterns, or configuration tuning that the ECC team implemented in `zcashd` to mitigate performance issues.
2. The summary explains the intended or observed effect of each identified change on `zcashd`'s performance.
3. The summary provides initial recommendations on which `zcashd` improvements seem most applicable to Zebra's architecture.
4. The analysis is documented in a markdown file and shared with the project team.

### Story 1.3 Triage of Performance-Related GitHub Issues
As a **Zebra Developer**, I want **a summary report of existing performance-related GitHub issues**, so that **common themes and potential instrumentation points can be identified.**
#### Acceptance Criteria
1. All open issues in the Zebra GitHub repository with the labels ["I-slow", "A-diagnostics", "I-heavy", or "I-unbounded-growth"](performance_related_issues.md) are reviewed.
2. The reviewed issues are categorized by the suspected area of the codebase (e.g., networking, state management, RPC, consensus).
3. A summary report is created that highlights any common themes or frequently reported problems.
4. The report is documented in a markdown file and shared with the project team.

### Story 1.4 Evaluation and Enhancement of Existing Metrics
As a **Zebra Developer**, I want to **evaluate the effectiveness of the current metrics and enhance them**, so that **we have the right level of visibility to diagnose performance issues.**
**Prerequisites:**
* Completion of **Story 1.1: DevOps Observation Environment Setup**, ensuring a usable Prometheus/Grafana pipeline and log indexing/viewing solution.
#### Acceptance Criteria
1. A comprehensive review of all metrics currently exposed by the `--metrics` listener is performed and documented.
2. The review identifies which existing metrics are useful for this project's goals and which key metrics are missing, based on the analysis from stories 1.2 and 1.3.
3. A prioritized list of metric additions or improvements is created and approved by the team.
4. The highest-priority additions or improvements are implemented in the Zebra codebase, following existing conventions.

### Story 1.5 Evaluation and Enhancement of Logging
As a **Zebra Developer**, I want to **evaluate and enhance structured logging in critical code paths**, so that **I can analyze execution flow and timing to diagnose bottlenecks.**
**Prerequisites:**
* Completion of **Story 1.1: DevOps Observation Environment Setup**, ensuring a usable log indexing/viewing solution.
* Completion of **Story 1.2 (`zcashd` Analysis)** and **Story 1.3 (Issue Triage)** to inform where to add logging.
#### Acceptance Criteria
1. A review of the current logging output is performed to identify gaps in coverage or detail, particularly concerning performance-sensitive operations.
2. A prioritized list of logging additions or improvements is created and approved by the team.
3. New structured log points (using the existing `tracing` infrastructure) are added to the highest-priority code paths (e.g., block verification, state updates, network message handling).
4. Logs related to performance must be structured (key-value pairs) and include context like timings and relevant identifiers to be easily parsable and searchable.

### Story 1.6 Operator-Facing Observability Documentation
As a **Node Operator**, I want **clear documentation for the new and enhanced observability features**, so that **I can effectively monitor my `zebrad` instance's performance.**
**Prerequisites:**
* Completion of **Story 1.4 (Metric Enhancement)** and **Story 1.5 (Logging Enhancement)**.
#### Acceptance Criteria
1. The documentation describes how to enable and configure the metrics endpoint and structured logging.
2. The documentation lists the key performance-related metrics and logs, explaining what they measure and how they can be interpreted.
3. A guide is provided on how to use the default Grafana dashboard (from NFR2) with a Prometheus data source to visualize Zebra's performance.
4. The documentation is added to the Zebra online book in an appropriate section.

---

## Epic 2 Performance Baselining & Bottleneck Identification
With the observability tools from Epic 1 in place, this epic focuses on systematic data collection and analysis. The engineering team will run the instrumented `zebrad` node under a variety of documented scenarios, establish a quantitative baseline for performance, and use this data to form specific, evidence-backed hypotheses about the primary software bottlenecks.

### Story 2.1 Baseline Sync Performance & "Sandblasting" Era Analysis
As a **Performance Engineer**, I want to **execute and document a baseline performance test of Zebra's initial sync**, so that **we have a benchmark for 'normal' operation.**
#### Acceptance Criteria
1. A `zebrad` node is synced from genesis on a target network (e.g., Mainnet) in the DevOps observation environment.
2. Key performance indicators (KPIs), such as total sync time, CPU/memory usage, disk I/O, IOPS, block processing latency, and network KPIs (e.g., inbound/outbound peer count), are collected throughout the sync.
3. The results are documented in a standardized report format.
4. This baseline is compared against an equivalent sync performed by `zcashd` to establish the initial performance gap.
5. The performance data collected during the sync portion between approximately block height **1,710,000 and 2,250,000** (the 'sandblasting' era) must be specifically isolated and analyzed to verify that performance does not materially degrade when processing these blocks, as compared to other periods.

### Story 2.2 Baseline Performance in Other Scenarios
As a **Performance Engineer**, I want to **execute and document performance tests for other key scenarios like high mempool utilization**, so that **we understand Zebra's behavior under various types of stress.**
**Prerequisites:**
* Completion of **Story 2.1**.
#### Acceptance Criteria
1. At least two additional stress-test scenarios are designed, focusing on conditions like high mempool utilization or rapid network peering fluctuations.
2. The tests are executed against a `zebrad` node in the DevOps observation environment.
3. The same set of Key Performance Indicators (KPIs) from Story 2.1 are collected throughout the tests.
4. The results are documented in the standardized report format, highlighting any performance degradation compared to the normal baseline.

### Story 2.3 Bottleneck Analysis and Hypothesis Formulation
As a **Zebra Developer**, I want to **analyze the data from all baseline tests**, so that **I can form specific, evidence-backed hypotheses about the primary software bottlenecks.**
**Prerequisites:**
* Completion of **Story 2.1** and **Story 2.2**.
#### Acceptance Criteria
1. All performance data collected from the previous stories in this epic is aggregated and analyzed.
2. The analysis identifies and ranks the most likely code paths or system interactions contributing to performance degradation.
3. For each identified bottleneck, a clear hypothesis is formulated explaining how it impacts performance.
4. A formal "Bottleneck Analysis Report" is created, summarizing the findings and hypotheses, to serve as the primary input for Epic 3.

---

## Epic 3 Targeted Remediation and Validation
This is the implementation phase. Based on the bottleneck analysis from Epic 2, this epic will consist of a series of technical stories focused on specific code remediations. Each fix will be developed and then validated using the same DevOps tooling and observability suite from Epic 1 to prove its effectiveness against the established baseline. The primary success criterion is achieving the project's main performance target.

### Story 3.1: Remediate Bottleneck #1 - [Description TBD]
As a **Zebra Developer**, I want to **implement the code changes to fix an identified bottleneck**, so that **Zebra's performance is measurably improved.**
**Prerequisites:**
* Completion of **Story 2.3: Bottleneck Analysis and Hypothesis Formulation**.
#### Acceptance Criteria
1. A code change that directly addresses a specific, high-priority bottleneck from the "Bottleneck Analysis Report" is implemented.
2. The change is accompanied by relevant unit and integration tests and passes all existing CI checks.
3. The change is validated in the DevOps observation environment, running the relevant test scenario(s).
4. The validation test shows a statistically significant improvement in the relevant Key Performance Indicators (e.g., reduced latency, lower CPU usage) compared to the established baseline.
5. The performance improvement is evaluated against the overall project goal of matching or exceeding `zcashd` performance.

---
*Note: Story 3.1 serves as a template. Multiple such stories will be created based on the findings from Epic 2.*

---

## Checklist Results Report

| Category | Status | Critical Issues |
| :--- | :--- | :--- |
| 1. Problem Definition & Context | PASS | None |
| 2. MVP Scope Definition | PASS | None |
| 3. User Experience Requirements | N/A | This project has no end-user UI. |
| 4. Functional Requirements | PASS | None |
| 5. Non-Functional Requirements | PASS | None |
| 6. Epic & Story Structure | PASS | None |
| 7. Technical Guidance | PASS | None |
| 8. Cross-Functional Requirements | PASS | None |
| 9. Clarity & Communication | PASS | None |
