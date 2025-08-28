---
layout: single
title: "deltagency"
subtitle: "open-source task-first AI agent framework"
excerpt: "open-source task-first AI agent framework"
toc: true
toc_label: "indice"
toc_icon: "heart"
toc_sticky: true
---

## execute, then delegate

while most AI agent frameworks today are conversation-obsessed and production-hostile, **deltagency** cuts through the noise by implementing a task-based, Ollama-native AI agent framework with zero bloat.  
in contrast to layers of abstractions of other frameworks, the goal is to provide a clean, simple interface to work with and make the workflow as deterministic as possible.

## vision

deltagency aims to provide:
- modular components instead of rigid hierarchies
- optimization for *input → output* workflows, not endless chat
- direct Ollama integration with full API control
- declarative YAML agent definitions
- minimal dependencies, maximum control

## architecture

deltagency is built with a simple but structured approach.  
its architecture combines clean design layers with a flexible relay pattern for orchestration.

### design layers

the core of deltagency is organized in clear layers.  
each layer has a single responsibility, making agents easier to build, debug, and extend.
- **communication bridge** → direct Ollama API integration
- **agent executor** → task execution
- **relay nodes** → composition of workflow made of multiple agents
- **relay runner** → the control loop that executes the nodes

### the relay pattern

traditional frameworks mix *execution* with *decision-making*, creating debugging nightmares  
deltagency introduces the **relay pattern**, which separates concerns:
- **execute** → agent completes its specialized task
- **delegate** → agent passes results or decides the next step
- **clean** → context is released, resources freed

```
# build any orchestration pattern
A ↔ B (consultation loop) → C (finalize)
Router → {SQL_Expert | XSS_Expert | Auth_Expert}
Coordinator ↔ {Worker1, Worker2, Worker3}
```

one pattern, infinite workflows

## why it’s different

most frameworks optimize for conversations or heavy orchestration.  
deltagency is different:
- **task-focused** → deterministic workflows, predictable outputs
- **modular** → composable, reusable components
- **ollama-native** → direct API integration, no wrappers
- **lightweight** → only requests, PyYAML, jsonschema
- **developer-friendly** → YAML configs and a clean API

## status

deltagency is under active development.  
early results:
- better modularity
- cleaner configurations
- equal accuracy with less overhead

code will be open-sourced once production stability is reached.

## in summary

deltagency is a lightweight, task-first framework for building modular AI agents.  
it focuses on deterministic workflows, direct Ollama integration, and minimal dependencies.  
the goal is simple: remove overhead and give developers predictable, controllable agents.

{% include project_sections.html project_name="deltagency" %}
