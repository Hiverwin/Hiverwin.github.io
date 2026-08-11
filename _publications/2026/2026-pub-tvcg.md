---
title:          "VisAgentBench: A Widget-Centric Framework and Benchmark for Autonomous Visual Analytics"
date:           2026-08-29 18:30:00 +0200
selected:       true
pub:            "(Under Review) IEEE Transactions on Visualization and Computer Graphics "
pub_date:       "2026"
abstract: >-

  Visual analytics (VA) enables sensemaking through interactive visualization, but effective analysis often requires experts to translate high-level intents into long sequences of interface operations and iteratively interpret visual feedback. We study whether modern visual-language models (VLMs) can take on this role as autonomous VA operators that observe the interface, plan multi-step exploration, execute interactions, and adapt based on intermediate visual feedback. To support systematic development and evaluation, we first introduce a widget-centric agentic VA framework that standardizes interactive components as structured widgets with unified action (e.g., filter and zoom) and perception-query (e.g., visible distributions, selection sum- maries) APIs. This standardization lets the framework wrap an existing Vega-based VA system to make it agent-operable without rebuilding it, or compose a new VA system by assembling widgets as reusable building blocks. To help agents coordinate across wid- gets rather than plan each interaction from scratch, each widget further packages reusable, chart-specific analytical workflows, giving agents more than a bare set of callable functions to plan over. Building on this framework, we present VisAgentBench, a benchmark of single- and multi-widget VA tasks that require agents to perform multi-step interactions to uncover evidence and produce verifiable results. Each task also provides fine-grained reference annotations, including an expected answer, an executable reference trajectory, and task-relevant state conditions, so VisAgentBench can score Answer, Reference Trace Similarity, and State separately rather than collapsing agent performance into one success score. Experiments across multiple VLMs show that our framework provides an effective scaffold for agentic VA, while the diagnostic measures expose persistent limitations in chart perception, operation grounding, state construction, and multi-step planning.

cover:  /assets/images/covers/acl.png
authors:
  - <strong>Yutong Chen</strong>*
  - Zhike Tang+
  - Zhihao Mai
  - Zhihao Shuai
  - Danli Luo
  - Jing Xu
  - Weikai Yang
#links:
#  Paper: https://aclanthology.org/2026.acl-long.1368/
#  PDF: /2026.acl-long.1368.pdf
---
