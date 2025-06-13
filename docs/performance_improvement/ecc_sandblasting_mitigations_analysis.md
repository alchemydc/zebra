# **zcashd Sandblasting Mitigations**

* [gDoc Version](https://docs.google.com/document/d/1fnKQ6RUs5hL5lj3aJ238Ag5esU7HsKXoZteceWcE3EI/edit?usp=sharing)
* [Audio Overview](https://drive.google.com/file/d/14LPN6K4U9NjIo1xS3oHrfDMr4HXs5NQa/view?usp=sharing)

## **Executive Summary**

Beginning in June 2022, the Zcash network was subjected to a sustained denial-of-service (DoS) campaign, colloquially termed the "sandblasting" attack. This attack did not exploit a cryptographic vulnerability but instead targeted computational and economic weaknesses within the zcashd client implementation. The attacker generated a high volume of transactions containing a large number of shielded outputs, maximizing the computational load on validating nodes while incurring minimal cost due to a fixed, low transaction fee structure.1 The effects were severe, leading to stalled node synchronization, unresponsive RPC endpoints, unusable light wallets, and significant blockchain bloat, effectively degrading the user experience and operational stability of the network.2

In response, the Electric Coin Co. (ECC) initiated a multi-phase mitigation strategy. This strategy can be broadly categorized into two distinct efforts. The first phase consisted of immediate, tactical code-level optimizations deployed across zcashd versions 5.1.0, 5.2.0, and 5.3.0. These changes focused on alleviating the acute CPU and memory pressure by implementing well-established performance engineering patterns, such as parallelizing zk-SNARK verification, batching cryptographic operations, and introducing validation caches to eliminate redundant computations.6 These measures were designed to restore node stability and keep the network operational under the ongoing stress.

The second phase represented a more strategic, long-term economic hardening of the network. This was primarily achieved through the implementation of Zcash Improvement Proposal (ZIP) 317, which introduced a proportional fee mechanism.10 Deployed as a node policy change in zcashd 5.5.0 rather than a consensus rule, this update recalibrated transaction fees to be proportional to the computational resources consumed, thereby removing the economic incentive for the sandblasting attack.12

**This report provides a detailed technical analysis of these code, architectural, and configuration changes, intended to serve as an actionable reference for the Zebra development team to inform its own client resilience and performance remediation strategies.**

## **1\. Symptom and Root Cause Analysis**

A comprehensive understanding of the mitigation strategies requires a precise diagnosis of the attack's effects and the underlying vulnerabilities in the zcashd client that the attacker exploited.

### **1.1 Observable Network and Node Symptoms**

The sandblasting attack manifested through a series of cascading performance failures across the Zcash ecosystem.

* **Node Performance Degradation:** Full node operators were the first to observe severe operational issues. Reports from the community and ECC developers indicated that zcashd instances were experiencing sustained 100% CPU utilization for extended periods.4 This led to extremely slow or completely stalled blockchain synchronization, particularly for nodes attempting to process blocks after height 1,700,000, which marked the period of intense spam activity. The  
  \-reindex command, a common recovery procedure, was similarly affected, becoming impractically slow.4  
* **RPC Unresponsiveness:** The high computational load had a direct impact on the node's JSON-RPC interface. Critical RPC methods, such as getblocktemplate used by mining pools, and other state-querying calls used by exchanges and services, would hang for long periods or time out entirely.4 Nodes would appear to be "locked up," processing a flurry of blocks quickly and then becoming unresponsive again, indicating severe contention for a shared resource.4  
* **Wallet and Light-Client Dysfunction:** For end-users, the most acute symptom was the failure of light-client infrastructure. Mobile wallets built on the ECC Software Development Kit (SDK), including Nighthawk, Edge, and Unstoppable, became effectively unusable.2 The massive influx of spam transactions created a "data pileup" that these wallets had to download and process.2 Due to Zcash's privacy model, wallets must perform a trial decryption on every shielded output on the chain to discover transactions belonging to the user. The spam transactions, with their hundreds of outputs, magnified this workload to an untenable degree, leading to extreme sync times and preventing users from accessing or spending their funds.2  
* **Blockchain Bloat:** A direct consequence of the attacker filling each block to its maximum 2 MB size was a rapid increase in the total size of the Zcash blockchain. Over a period of a few months starting in mid-2022, the chain size nearly tripled, growing from approximately 30-31 GB to over 100 GB.1 This increased the storage and bandwidth requirements for all full node operators.

### **1.2 Root Cause Analysis: A Confluence of Factors**

The performance degradation was not due to a single flaw but rather a combination of computational, architectural, and economic vulnerabilities in zcashd.

* **Attack Vector:** The malicious actor exploited Zcash's shielded transaction protocol by crafting transactions with an abnormally high number of outputs (for both the Sapling and Orchard pools).3 This technique, identified as a practical implementation of the theoretical "woodchipper attack," was designed to maximize the computational and storage burden per transaction.15  
* **Computational Bottleneck in zk-SNARK Verification:** The primary performance bottleneck was the computationally intensive process of verifying the zero-knowledge proofs (zk-SNARKs) associated with each shielded spend and output.3 Before the attack, the zcashd validation logic processed these proofs serially, one by one. For a transaction with hundreds of outputs, this created a massive computational workload that a single CPU core struggled to handle in a timely manner. The lack of standard optimizations like batch verification, where multiple proofs are checked together in a more efficient single operation, made the node highly susceptible to this load.6  
* **Architectural Bottleneck: cs\_main Lock Contention:** A deeper, architectural root cause was identified by ECC developers as contention on the cs\_main global lock, a legacy component from the Bitcoin Core codebase from which zcashd was forked.4 This critical section controls all read and write access to the global chain state. The main message handling thread would acquire this lock for the entire duration of block validation. Because the spam blocks took so long to validate due to the computational bottleneck, cs\_main was held for extended periods. Simultaneously, any incoming RPC call that needed to read the chain state (e.g., getinfo, getblocktemplate) would also attempt to acquire cs\_main and would be blocked, leading to the observed RPC unresponsiveness and timeouts. This demonstrates that the attack exploited not just an inefficient algorithm but a fundamental architectural limitation that prevented concurrent processing of validation and RPC requests.  
* **Economic Vulnerability due to Fixed Fees:** The attack was economically feasible due to Zcash's fixed transaction fee of 1,000 zatoshi (0.00001 ZEC), which was not tied to transaction size or complexity.10 This allowed the attacker to fill blocks with computationally expensive transactions for an estimated cost of only $10 per day.1 The absence of a fee market or a proportional fee mechanism, a known theoretical risk 3, created a situation where there was no economic disincentive to prevent this resource exhaustion attack.  
* **Memory Pressure and Vulnerabilities:** The attack also induced significant memory pressure. While a distinct memory exhaustion vulnerability inherited from Bitcoin Core was discovered and patched by Halborn in parallel 18, the sandblasting attack itself created memory issues. In particular, after initial wallet scanning optimizations were introduced, some nodes began to experience Out-Of-Memory (OOM) aborts due to the memory usage of the new batch scanner, necessitating further fixes.9

The confluence of these factors created a perfect storm. The attacker used an inexpensive economic vulnerability to trigger a computational bottleneck, which in turn exacerbated a core architectural bottleneck, leading to a network-wide degradation of service. The fixes implemented by ECC were a direct response to this chain of dependencies, addressing not just the surface-level symptoms but also the underlying technical and economic debt. The Zebra client, having been designed with a more modern Rust-based architecture, is likely less exposed to the specific cs\_main lock contention issue. However, the sandblasting attack serves as a critical case study, demonstrating that any unoptimized, computationally-intensive code path can become a security vulnerability when subjected to a targeted resource exhaustion attack.

## **2\. Code-Level Changes and Mitigations**

In the immediate aftermath of the attack, ECC's primary focus was on tactical, code-level optimizations within the zcashd client. These changes, rolled out across versions 5.1.0 to 5.3.0, were designed to stabilize the network by directly addressing the most severe performance bottlenecks. This effort represented a significant paying down of technical debt in the client's validation and wallet subsystems.

### **2.1 Immediate Performance Optimizations (Releases 5.1.0 \- 5.3.0)**

This first phase of the response involved implementing standard, yet previously absent, performance engineering patterns.

#### **2.1.1 Parallelism and Batch Validation (Release 5.1.0)**

The single most impactful change was the move from serial, one-by-one verification of cryptographic components to parallelized batch validation. This technique amortizes the high fixed cost of verification setup across many items, dramatically improving throughput.

* **Implementation:** The changes were introduced in zcashd 5.1.0 and targeted both the Orchard and Sapling shielded pools.  
  * **Orchard Pool:** Halo 2 proofs for Orchard Actions, which were the initial focus of the spam, were refactored to be checked via batch validation and multithreading.7 This work was contained in **Pull Request \#6023**.4  
  * **Sapling Pool:** As the attacker shifted to spamming Sapling transactions, ECC quickly followed with **Pull Request \#6048**, which implemented batch validation for Sapling's Groth16 zk-SNARK proofs and parallelized verification for RedJubjub signatures.4  
* **Impact:** ECC reported that these changes collectively resulted in a reduction of worst-case block validation times by approximately 80% on their benchmark hardware (a Ryzen 9 5950X CPU).2 This was a critical first step in reducing the time the cs\_main lock was held, thereby improving overall node health.  
* **Configuration:** The degree of parallelism for these operations is not controlled by a zcash.conf parameter but by the RAYON\_NUM\_THREADS environment variable, which is standard for applications using the Rayon parallel computing library in Rust.7

#### **2.1.2 Introduction of Validation Caching (Release 5.2.0)**

To further reduce computational overhead and cs\_main lock contention, ECC introduced a caching layer to avoid re-validating transactions that had already been verified upon entry to the mempool.

* **Implementation:** This was accomplished in **Pull Request \#6073** for the 5.2.0 release.4 The implementation added two new CuckooCache instances to the validation module. These probabilistic data structures efficiently store the validation status of transaction bundles (proofs and signatures). When a block is received, transactions that are found in the cache are not re-submitted to the expensive batch validators, effectively making their final validation a no-op.4  
* **Impact:** This change significantly sped up the processing of blocks where most transactions were already known to the node via mempool gossip. This directly reduced the duration of the ActivateBestChain step, alleviating the RPC unresponsiveness caused by the cs\_main lock.4

#### **2.1.3 Wallet-Side Scanning Optimizations and Memory Management (Releases 5.2.0 & 5.3.0)**

While the above changes stabilized the core node, the zcashd internal wallet and the dependent light-client SDKs remained a major pain point.

* **Parallel Trial Decryption (v5.2.0):** To address the slow wallet sync times, the process of trial-decrypting Sapling outputs to scan for incoming notes was parallelized. Instead of a linear scan, outputs were processed in parallel batches, significantly improving performance for wallet users.6  
* **Memory Management (v5.3.0):** A negative consequence of the new parallel scanner was that on some systems, it led to uncontrolled memory growth, causing the zcashd process to be terminated by the operating system (OOM abort).9 Release 5.3.0 addressed this by refactoring the batch scanner to be more memory-efficient and, crucially, by imposing a hard memory limit of  
  **100 MiB** on its operation, preventing future OOM conditions.9

The following table summarizes the key pull requests that constituted this initial, code-level response. For the Zebra team, these PRs point to the specific areas of the zcashd codebase that were most critical to fortify against this type of attack.

| PR Number | Title/Summary | Target Release | Technical Change & Significance | Snippet Refs |
| :---- | :---- | :---- | :---- | :---- |
| **\#6023** | Batch-verify Orchard proofs | 5.1.0 | Introduced multithreaded batch validation for Halo 2 proofs in the Orchard pool. The first major step in tackling the computational bottleneck. | 4 |
| **\#6048** | Batch-verify Sapling proofs and signatures | 5.1.0 | Extended batch validation and multithreading to Sapling's Groth16 proofs and RedJubjub signatures. Addressed the attacker's shift to Sapling spam. | 4 |
| **\#6073** | Cache Sapling and Orchard bundle validation | 5.2.0 | Implemented two CuckooCaches to store validation results, preventing re-verification of transactions already seen in the mempool. Reduced contention on the cs\_main lock. | 4 |
| **(Implied)** | Parallelize trial decryption of Sapling outputs | 5.2.0 | Modified the internal zcashd wallet to scan for incoming notes in parallel batches, improving wallet sync performance. | 6 |
| **\#6192** (related) | Reduce steady-state memory utilization | 5.3.0 | Addressed OOM issues caused by the new batch scanner by optimizing memory usage and imposing a 100 MiB limit. | 9 |

## **3\. Architectural and Economic Hardening**

While the initial code optimizations were critical for network survival, they did not address the fundamental economic vulnerability that made the sandblasting attack so cheap and effective. The second phase of ECC's response focused on strategic changes to the node's economic policies and configurable behaviors.

### **3.1 The Proportional Fee Mechanism (ZIP-317)**

The sandblasting attack underscored the unsustainability of a low, fixed transaction fee in a system with variable computational costs.10 The long-term solution was the development and deployment of a new fee policy, specified in ZIP-317, which aimed to make transaction fees proportional to the network resources they consume.11

#### **3.1.1 Specification and Rationale**

The core innovation of ZIP-317 is the concept of "logical actions" and a fee formula based upon them.

* Logical Actions: Instead of basing fees on transaction size in bytes, which does not accurately reflect the computational cost of zk-SNARK verification, ZIP-317 defines a transaction's cost in terms of its logical components. The number of logical actions is calculated as:  
  logicala​ctions=max(Ntin​​,Ntout​​)+max(Nsspend​​,Nsout​​)+Noaction​​  
  where Ntin/out​​ are the number of transparent inputs/outputs, Nsspend/out​​ are the number of Sapling spends/outputs, and Noaction​​ is the number of Orchard actions.10 This metric ensures that the fee scales with the most resource-intensive parts of a transaction.  
* Fee Formula: The conventional transaction fee is then calculated using this metric:  
  $fee \= \\text{base\_fee} \+ \\text{marginal\_fee} \\times \\max(0, \\text{logical\_actions} \- \\text{grace\_actions})$  
  The specific parameters defined were 10:  
  * base\_fee: 10,000 zatoshi  
  * marginal\_fee: 5,000 zatoshi  
  * grace\_actions: 2

This structure ensures that a standard privacy-preserving transaction (e.g., 2 shielded actions) pays only the base fee, while larger transactions, like those used in the spam attack, incur a proportionally higher cost. The grace window of 2 actions was specifically included to avoid penalizing users for standard wallet behavior that pads transactions to 2 inputs and 2 outputs to reduce metadata leakage.11

#### **3.1.2 Implementation as a Policy Change (Release 5.5.0)**

A crucial aspect of the ZIP-317 deployment was its implementation as a set of node policy changes rather than as a network-wide consensus rule change.10 This approach offered significantly more agility. A consensus change would have required a hard fork, a slow and complex process involving coordination across the entire ecosystem. By implementing ZIP-317 as a policy within zcashd, ECC could deploy the fix in a standard software release. Nodes running version 5.5.0 and later would begin to prioritize transactions in their mempool and in the blocks they mine based on the new fee rules. This created a powerful economic incentive for users and other miners to adopt the new fee structure to ensure their transactions were processed, driving adoption without forcing a disruptive network upgrade.

#### **3.1.3 The "Unpaid Actions" System and blockunpaidactionlimit**

To manage the transition and prevent the new policy from breaking existing wallets that were not yet ZIP-317 aware, zcashd introduced a mechanism to handle transactions that did not meet the new conventional fee.10

* **Unpaid Action Quota:** A small quota of "unpaid actions" was reserved in each block. Transactions with fees below the new conventional fee would compete for this limited space, with preference given to those that were "less unpaid".10 This provided a probabilistic path for legacy transactions to still be mined, albeit more slowly.  
* **blockunpaidactionlimit Parameter:** This policy was controlled by the blockunpaidactionlimit configuration parameter, which was introduced for miners. It defaulted to 50, allowing for a total of 50 unpaid logical actions per block template.10 This served as a critical safety valve during the transition period.  
* **End of the Grace Period:** As the ecosystem of wallets and services adapted to the new fee structure, this transitional support was phased out. In zcashd version 6.1.0, the default for blockunpaidactionlimit was changed to zero, effectively requiring all transactions to adhere to the ZIP-317 fee structure for reliable inclusion in blocks mined by default-configured nodes.24

### **3.2 New and Relevant Configuration Parameters**

The sandblasting attack and the subsequent mitigations highlighted the importance of node configurability for performance tuning and defense. The following table summarizes the key parameters that became relevant during this period. For the Zebra team, this provides a reference for the set of controls that zcashd operators now have at their disposal.

| Parameter / Variable | Type | Purpose | Relevant Release(s) | Recommended Setting / Default | Snippet Refs |
| :---- | :---- | :---- | :---- | :---- | :---- |
| RAYON\_NUM\_THREADS | Environment Var | Controls the number of threads used for parallel proof validation. | 5.1.0 | Set to the number of available CPU cores for maximum performance. | 7 |
| mempooltxcostlimit | zcash.conf | Sets the total "cost" limit for the mempool, triggering eviction when exceeded. | 2.1.0-1 (pre-attack) | Default: 80,000,000. A key parameter for limiting mempool bloat. | 31 |
| Batch Scanner Memory | Hardcoded Limit | Limits the wallet's batch scanner to 100 MiB to prevent OOM aborts. | 5.3.0 | 100 MiB (not user-configurable). | 9 |
| blockunpaidactionlimit | zcash.conf | For miners: sets the per-block limit for "unpaid" logical actions under ZIP-317. | 5.5.0 | Initially 50; changed to 0 in v6.1.0. Recommended: 0\. | 10 |
| dbcache | zcash.conf | Sets the database cache size in MB. Not new, but highlighted as critical for sync performance. | Pre-existing | Default: 450\. Recommended: Increase significantly (e.g., 8000 for 8GB) on systems with high RAM to dramatically speed up IBD. | 32 |

## **4\. Validation, Monitoring, and Impact Assessment**

Evaluating the effectiveness of the implemented changes and establishing robust monitoring are crucial components of a complete incident response cycle.

### **4.1 Measuring Efficacy and Impact**

The success of ECC's mitigations was measured through a combination of quantitative benchmarks and qualitative network observation.

* **Quantitative Metrics:** The most frequently cited metric for the initial code-level optimizations was the **\~80% reduction in worst-case block validation times**.6 This was measured internally by ECC on a specific high-end CPU (Ryzen 9 5950X) and served as the primary benchmark for the effectiveness of the parallelism and batching changes in zcashd 5.1.0. While this is a powerful indicator of improvement in the core computational bottleneck, there is a notable lack of publicly available, comprehensive performance reports or charts from ECC that demonstrate the "before and after" impact across a wider range of hardware and network conditions.25 The Ziggurat network testing framework exists to perform conformance, performance, and resistance testing, but its published results are high-level and do not provide detailed performance benchmarks for this specific incident.28  
* **Qualitative Impact of ZIP-317:** The impact of the proportional fee mechanism was primarily assessed qualitatively. Community discussion and ECC retrospective reports indicate that the activation of the ZIP-317 policy in zcashd 5.5.0 had an "immediate effect" on the nature of the spam.10 By substantially increasing the cost to fill blocks with high-action transactions, the attack became economically less viable. The spam campaign ultimately ceased entirely in November 2023, approximately six months after the ZIP-317 policy was widely deployed.15

### **4.2 New Observability Hooks: Prometheus Metrics**

A key lesson from the sandblasting attack was the need for better real-time visibility into the internal state of the zcashd client. In response, ECC added several new application-specific metrics exposed via the \-prometheusport configuration, allowing node operators to monitor performance and network health with much greater precision.

* **Release 5.3.0 (Wallet Scanner Metrics):** This release introduced metrics specifically for monitoring the wallet's batch scanning process, which had been a source of both performance bottlenecks and memory exhaustion.  
  * zcashd.wallet.batchscanner.outputs.scanned (counter): Tracks the total number of shielded outputs processed by the scanner.  
  * zcashd.wallet.batchscanner.size.transactions (gauge): Shows the current number of transactions queued in the scanner.  
  * zcashd.wallet.batchscanner.usage.bytes (gauge): Reports the current memory footprint of the batch scanner.  
  * zcashd.wallet.synced.block.height (gauge): Indicates the block height to which the wallet is synced.  
  * **Significance:** This suite of metrics provides direct insight into the workload, queue depth, and memory pressure of the wallet scanning subsystem, allowing operators to diagnose wallet-specific performance issues that were central to the user-facing problems during the attack.9  
* **Release 5.8.0 (Mempool Fee Metrics):** Following the deployment of ZIP-317, new metrics were added to monitor the economic composition of the mempool.  
  * mempool.actions.unpaid: A histogram detailing the distribution of unpaid logical actions in mempool transactions.  
  * mempool.actions.paid: A counter for the total number of paid logical actions in the mempool.  
  * mempool.size.weighted: A histogram of transaction "costs" (as defined in ZIP-401) in the mempool.  
  * **Significance:** These metrics are essential for observing the real-time effects of the ZIP-317 fee policy. They allow operators to see the proportion of paid vs. unpaid transactions, detect influxes of low-fee spam, and monitor the overall economic health of the transaction relay network.24

The introduction of these metrics, while a significant improvement, was largely a reactive measure. The most critical, targeted metrics for diagnosing the attack's impact on the wallet and mempool were added months after the attack began. This suggests that during the initial crisis, engineers had to rely on more generic system-level tools (like CPU profilers) rather than purpose-built application metrics. This underscores the critical importance of building comprehensive, proactive observability into a client's design from the outset, as it enables a much faster and more precise response to novel threats.

## **5\. Actionable Insights and Recommendations for the Zebra Team**

The Zcash network's experience with the sandblasting attack and ECC's subsequent mitigation efforts in zcashd provide a rich set of learnings. This analysis can be synthesized into a series of actionable recommendations for the Zebra development team to enhance the resilience and performance of its own Zcash node implementation.

### **5.1 Key Takeaways from the zcashd Response**

The zcashd case study reveals several core principles for building robust blockchain clients in an adversarial environment.

* **Resource Exhaustion is a Primary Attack Vector:** The attack demonstrated that computationally correct but unoptimized code paths are a significant security vulnerability. An attacker can leverage these hotspots to degrade or deny service without needing to break any cryptographic primitives.  
* **A Multi-layered Defense is Essential:** The most effective response combined two distinct strategies: immediate, tactical code optimization (parallelism, caching) to ensure short-term survival, followed by long-term, strategic economic hardening (proportional fees) to remove the attacker's incentive. Neither approach would have been sufficient on its own.  
* **Policy-Based Mitigations Offer Agility:** The decision to implement ZIP-317 as a node-level policy, rather than a consensus rule, was a key strategic success. It allowed for rapid deployment and adaptation, using economic incentives to drive network-wide adoption without the high coordination cost and risk of a hard fork.  
* **Architectural Simplicity and Modern Concurrency are Strengths:** A significant portion of zcashd's performance degradation was attributable to its monolithic cs\_main global lock, an architectural artifact from its Bitcoin Core lineage. This highlights that a modern concurrency model, which avoids such global bottlenecks, is a strategic advantage in preventing cascading failures under load.  
* **Proactive Observability is Non-Negotiable:** The most useful, application-specific metrics for diagnosing the sandblasting attack were added to zcashd reactively. This underscores the need to instrument a client proactively with detailed metrics for all critical subsystems, as this is a prerequisite for rapid and effective incident response.

### **5.2 Strategic Recommendations for Zebra's Remediation and Resilience Strategy**

Based on these takeaways, the following strategic recommendations are proposed for the Zebra development team.

* **Recommendation 1: Conduct a Targeted Performance and Concurrency Audit.**  
  * **Action:** Undertake a systematic audit of Zebra's entire transaction validation pipeline, from P2P message deserialization through script execution and final block connection. Use profiling tools to identify and benchmark all computationally expensive operations, particularly zk-SNARK proof and signature verification. Pay special attention to the concurrency model. Learning from the cs\_main bottleneck in zcashd, critically analyze Zebra's locking strategies to ensure that no single lock or sequential process can become a global bottleneck under high load, which would make the client vulnerable to similar RPC-stalling attacks.  
* **Recommendation 2: Implement and Enhance ZIP-317 Logic.**  
  * **Action:** Prioritize the full implementation of the ZIP-317 proportional fee mechanism as a mempool and block production policy, mirroring the successful approach taken by zcashd.29 This is the most potent long-term defense against economic spam. Furthermore, the team should consider potential future enhancements. For example, could the fee parameters (  
    base\_fee, marginal\_fee) be made dynamic, adjusting automatically based on sustained network load or block fullness? This could evolve the policy into a more robust, EIP-1559-style fee market, further increasing network resilience.  
* **Recommendation 3: Develop a "Sandblasting" Simulation Framework.**  
  * **Action:** Design and build an internal testing framework capable of simulating the sandblasting attack at scale. This tool should generate a high volume of transactions with a large number of shielded actions and submit them to a testnet running Zebra. By integrating this framework into the continuous integration and deployment (CI/CD) pipeline, the team can run regular, automated stress tests. This will proactively identify performance regressions and new bottlenecks before they can be exploited on the mainnet, shifting Zebra's security posture from reactive to proactive.  
* **Recommendation 4: Implement a Comprehensive Metrics and Alerting Suite.**  
  * **Action:** Proactively instrument the Zebra codebase with a rich set of Prometheus metrics that provide deep visibility into all critical subsystems. This suite should, at a minimum, be equivalent to the metrics ECC added to zcashd post-attack 9, covering mempool composition (transaction costs, fee levels, unpaid actions), validation and scanning queue depths, and cache hit/miss rates. This detailed observability should be coupled with an automated alerting system (e.g., via Alertmanager) to notify the development team of anomalous network activity in real-time, drastically reducing the time required to detect, diagnose, and respond to future incidents.

#### **Works cited**

1. Zcash continues to suffer from spam attack that started months ago, accessed June 13, 2025, [https://www.web3isgoinggreat.com/single/zcash-continues-to-suffer-from-spam-attack-that-started-months-ago](https://www.web3isgoinggreat.com/single/zcash-continues-to-suffer-from-spam-attack-that-started-months-ago)  
2. A look back: NU5 and network sandblasting \- Electric Coin Company, accessed June 13, 2025, [https://electriccoin.co/blog/a-look-back-nu5-and-network-sandblasting/](https://electriccoin.co/blog/a-look-back-nu5-and-network-sandblasting/)  
3. Someone is clogging up the Zcash blockchain with a spam attack | The Block, accessed June 13, 2025, [https://www.theblock.co/post/175259/someone-is-clogging-up-the-zcash-blockchain-with-a-spam-attack](https://www.theblock.co/post/175259/someone-is-clogging-up-the-zcash-blockchain-with-a-spam-attack)  
4. Very slow sync and \`-reindex\` after 28.16.2022 · Issue \#6049 · zcash/zcash \- GitHub, accessed June 13, 2025, [https://github.com/zcash/zcash/issues/6049](https://github.com/zcash/zcash/issues/6049)  
5. Disappointed with wallet experiences lately \- Page 2 \- Zcash Apps, accessed June 13, 2025, [https://forum.zcashcommunity.com/t/disappointed-with-wallet-experiences-lately/42442?page=2](https://forum.zcashcommunity.com/t/disappointed-with-wallet-experiences-lately/42442?page=2)  
6. Zcash Blockchain Size—Risks? \- General, accessed June 13, 2025, [https://forum.zcashcommunity.com/t/zcash-blockchain-size-risks/43126](https://forum.zcashcommunity.com/t/zcash-blockchain-size-risks/43126)  
7. New Release 5.1.0 \- Electric Coin Company, accessed June 13, 2025, [https://electriccoin.co/blog/new-release-5-1-0/](https://electriccoin.co/blog/new-release-5-1-0/)  
8. New Release 5.2.0 \- Electric Coin Company, accessed June 13, 2025, [https://electriccoin.co/blog/new-release-5-2-0/](https://electriccoin.co/blog/new-release-5-2-0/)  
9. New release 5.3.0 \- Electric Coin Company, accessed June 13, 2025, [https://electriccoin.co/blog/new-release-5-3-0/](https://electriccoin.co/blog/new-release-5-3-0/)  
10. Understanding why transactions are taking longer to be mined and ..., accessed June 13, 2025, [https://forum.zcashcommunity.com/t/understanding-why-transactions-are-taking-longer-to-be-mined-and-what-is-going-on-with-the-mempool/45399](https://forum.zcashcommunity.com/t/understanding-why-transactions-are-taking-longer-to-be-mined-and-what-is-going-on-with-the-mempool/45399)  
11. zips/zips/zip-0317.rst at main · zcash/zips \- GitHub, accessed June 13, 2025, [https://github.com/zcash/zips/blob/main/zips/zip-0317.rst](https://github.com/zcash/zips/blob/main/zips/zip-0317.rst)  
12. New Release 5.5.0 \- Electric Coin Company, accessed June 13, 2025, [https://electriccoin.co/blog/new-release-5-5-0/](https://electriccoin.co/blog/new-release-5-5-0/)  
13. Zcash Creator Releases v5.5.0, Latest Software Upgrade Now Live ..., accessed June 13, 2025, [https://www.securities.io/zcash-creator-releases-v5-5-0-latest-software-upgrade-now-live/](https://www.securities.io/zcash-creator-releases-v5-5-0-latest-software-upgrade-now-live/)  
14. High transaction volumes causing problems for miners · Issue \#2333 \- GitHub, accessed June 13, 2025, [https://github.com/zcash/zcash/issues/2333](https://github.com/zcash/zcash/issues/2333)  
15. arboretum-notes/AllArboristCallNotes/Sandblasting Retrospective \- Summary.md at main \- GitHub, accessed June 13, 2025, [https://github.com/ZcashCommunityGrants/arboretum-notes/blob/main/AllArboristCallNotes/Sandblasting%20Retrospective%20-%20Summary.md](https://github.com/ZcashCommunityGrants/arboretum-notes/blob/main/AllArboristCallNotes/Sandblasting%20Retrospective%20-%20Summary.md)  
16. Zcash chain triples in size thanks to $10-a-day spam attack \- Protos, accessed June 13, 2025, [https://protos.com/zcash-chain-triples-in-size-thanks-to-10-a-day-spam-attack/](https://protos.com/zcash-chain-triples-in-size-thanks-to-10-a-day-spam-attack/)  
17. Zcash Network Suffers Spam Attack of Shielded Transactions | News \- ihodl.com, accessed June 13, 2025, [https://ihodl.com/topnews/2022-10-06/zcash-network-suffers-spam-attack-shielded-transactions/](https://ihodl.com/topnews/2022-10-06/zcash-network-suffers-spam-attack-shielded-transactions/)  
18. New releases remediate memory exhaustion vulnerability in Zcash \- Electric Coin Company, accessed June 13, 2025, [https://electriccoin.co/blog/new-releases-remediate-memory-exhaustion-vulnerability-in-zcash/](https://electriccoin.co/blog/new-releases-remediate-memory-exhaustion-vulnerability-in-zcash/)  
19. Final Copy: ZF Q1 Report \- Zcash Foundation, accessed June 13, 2025, [https://zfnd.org/wp-content/uploads/2023/05/Zcash-Foundation-Q1-2023-Report.pdf](https://zfnd.org/wp-content/uploads/2023/05/Zcash-Foundation-Q1-2023-Report.pdf)  
20. All ECC teams focused on wallet performance \- Page 2 \- Ecosystem Updates, accessed June 13, 2025, [https://forum.zcashcommunity.com/t/all-ecc-teams-focused-on-wallet-performance/42860?page=2](https://forum.zcashcommunity.com/t/all-ecc-teams-focused-on-wallet-performance/42860?page=2)  
21. ZCash is under attack \- Cypherpunk Times, accessed June 13, 2025, [https://www.cypherpunktimes.com/zcash-is-under-attack/](https://www.cypherpunktimes.com/zcash-is-under-attack/)  
22. (Technical): Why does \`batch::try\_compact\_note\_decryption\` need a \`SaplingDomain\` for each Output? \- Zcash Community Forum, accessed June 13, 2025, [https://forum.zcashcommunity.com/t/technical-why-does-batch-try-compact-note-decryption-need-a-saplingdomain-for-each-output/42540](https://forum.zcashcommunity.com/t/technical-why-does-batch-try-compact-note-decryption-need-a-saplingdomain-for-each-output/42540)  
23. Recommended ZEC transaction fee settings? \- General \- Zcash Community Forum, accessed June 13, 2025, [https://forum.zcashcommunity.com/t/recommended-zec-transaction-fee-settings/49151](https://forum.zcashcommunity.com/t/recommended-zec-transaction-fee-settings/49151)  
24. Releases · zcash/zcash \- GitHub, accessed June 13, 2025, [https://github.com/zcash/zcash/releases](https://github.com/zcash/zcash/releases)  
25. accessed December 31, 1969, [https.github.com/zcash/zcash/pull/6023](http://docs.google.com/https.github.com/zcash/zcash/pull/6023)  
26. accessed December 31, 1969, [https.github.com/zcash/zcash/pull/6048](http://docs.google.com/https.github.com/zcash/zcash/pull/6048)  
27. accessed December 31, 1969, [https.github.com/zcash/zcash/pull/6073](http://docs.google.com/https.github.com/zcash/zcash/pull/6073)  
28. runziggurat/zcash: The Zcash Network Stability Framework \- GitHub, accessed June 13, 2025, [https://github.com/runziggurat/zcash](https://github.com/runziggurat/zcash)  
29. Zcash Foundation Q2 2023 Report, accessed June 13, 2025, [https://zfnd.org/wp-content/uploads/2023/09/Zcash-Foundation-Q2-2023-Report.pdf](https://zfnd.org/wp-content/uploads/2023/09/Zcash-Foundation-Q2-2023-Report.pdf)  
30. Peak memory usage during initial block download causes OOM on 4GB machine \#6268, accessed June 13, 2025, [https://github.com/zcash/zcash/issues/6268](https://github.com/zcash/zcash/issues/6268)  
31. ZIP 401: Addressing Mempool Denial-of-Service, accessed June 13, 2025, [https://zips.z.cash/zip-0401](https://zips.z.cash/zip-0401)  
32. Tips on zcashd sync? \- Technical Support \- Zcash Community Forum, accessed June 13, 2025, [https://forum.zcashcommunity.com/t/tips-on-zcashd-sync/45534](https://forum.zcashcommunity.com/t/tips-on-zcashd-sync/45534)  
33. Bitcoin full validation sync performance \- Casa Blog, accessed June 13, 2025, [https://blog.casa.io/bitcoin-full-validation-sync-performance/](https://blog.casa.io/bitcoin-full-validation-sync-performance/)