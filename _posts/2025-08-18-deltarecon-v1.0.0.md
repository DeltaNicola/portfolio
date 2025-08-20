---
layout: single
title:  "deltarecon - v1.0.0 haboob"
excerpt: "autonomous penetration testing meets artificial intelligence"
date:   2025-08-17 16:01:18 +0200
categories: deltarecon
tags: [deltarecon, release]
toc: true
toc_label: "indice"
toc_icon: "heart"
---

## objective
create an AI-powered penetration testing automation platform that handles everything from domain discovery to exploit generation, leveraging AI to its fullest potential.

## main features of the release

### complete workflow
the platform is structured in 5 main phases:

1. **setup** - automatic tool installation
2. **domain discovery** - domain and subdomain discovery 
3. **reconnaissance** - http/https technology scanning
4. **scanning** - vulnerability analysis (webapp + infrastructure)
5. **ai evaluation** - multi-agent intelligent analysis

### AI innovations
- **multi-AI crews**: system of 3 specialized crews
  - analyst crew (2 agents) for vulnerability analysis
  - formatting crew (1 agent) for output formatting  
  - correlation crew (1 agent) for exploit correlations
- **multiple LLM support via Ollama**: qwen, deepseek, mixtral, llama, mistral

### configuration features
- **YAML-based**: pipeline and playbook configurable via YAML
- **modular scans**: webapp/infra/full customizable
- **flexible LLM management**: different models for specific tasks
- **flexible targets**: automatic or predefined domains
- **gpu on-demand**: automatic resource management

### modern architecture
- **state saving** with etcd for automatic recovery
- **distributed storage** with minio for json results
- **automated deployment** via ansible
- **integrated notifications** via discord for monitoring
- **cloud management** runpod for GPU resources

## added value

this release represents a sophisticated platform that combines:
- **traditional tools** of consolidated pentesting
- **modern AI** for intelligent automation  
- **correlated analysis** of vulnerabilities
- **cloud scalability** with automatic resource management
- **enterprise configurability** for complex environments

the "why not?" philosophy is reflected in the integration of cutting-edge technologies to create a next-generation pentesting tool.