---
title: "Publications"
---

# Publications

This page lists my latest publications, research articles, and selected academic work.

<div class="card-base border border-[var(--border)] p-8 rounded-[var(--radius-large)] mb-6">

### [Q1 (Highest) – Top 5 AI Journals] AutoWAFuzzer: An adaptive framework for web application firewall penetration testing with multi-agent system and RAG-enabled reinforcement learning

**Expert Systems With Applications (Elsevier)** · April 28, 2026  
**DOI:** [10.1016/j.eswa.2026.132546](https://doi.org/10.1016/j.eswa.2026.132546)

<details class="group mt-4 rounded-[var(--radius-medium)] border border-[var(--border)] bg-[var(--card-bg)] p-5">
<summary class="cursor-pointer text-[var(--primary)] font-semibold">Read more</summary>

Web Application Firewalls (WAFs) are crucial in mitigating web-based threats such as SQLi and XSS, yet the evolving complexity of WAF detection mechanisms poses significant challenges for penetration testing (pentest) tools.

Existing ML- and RL-based fuzzers often suffer from three main limitations: (1) reliance on static training datasets, making them unflexible to new WAF rules; (2) monolithic single-agent architectures, which hinder diverse strategy exploration; and (3) lack of contextual awareness due to missing integration with real-world threat intelligence.

To address these challenges, we propose AutoWAFuzzer, an adaptive multi-agent framework that integrates Large Language Models (LLM), Reinforcement Learning (RL), and Retrieval-Augmented Generation (RAG). AutoWAFuzzer decomposes the testing process into modular agents: a generative LLM agent, an RL-based policy optimizer, a Reward Model agent simulating WAF feedback, and a RAG agent that continuously retrieves threat context from structured sources like MISP.

This design enables parallel strategy exploration, semantic conditioning of payloads, and continuous policy refinement in a closed feedback loop. Experimental evaluations across rule-based and ML-based WAFs—including ModSecurity, Naxsi, WAF-Brain, and CloudGuard—demonstrate that AutoWAFuzzer significantly outperforms prior approaches in bypass rate, adaptability, and generalization, advancing the state-of-the-art in automated WAF penetration testing.

</details>

</div>

<div class="card-base border border-[var(--border)] p-8 rounded-[var(--radius-large)] mb-6">

### test

**test** · test  
**DOI:** [test](test)

<details class="group mt-4 rounded-[var(--radius-medium)] border border-[var(--border)] bg-[var(--card-bg)] p-5">
<summary class="cursor-pointer text-[var(--primary)] font-semibold">Read more</summary>

test

</details>

</div>
