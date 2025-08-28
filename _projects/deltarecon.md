---
layout: single
title: "deltarecon"
subtitle: "AI-powered recon & pentesting engine"
excerpt: "AI-powered recon & pentesting engine"
toc: true
toc_label: "indice"
toc_icon: "heart"
toc_sticky: true
---
## AI-cybersec

**deltarecon** is a project born to transform reconnaissance and vulnerability assessment work into a **completely automated** process, scalable and supported by artificial intelligence  
the goal is to integrate all bug bounty phases: from asset collection to scanning, up to reporting in a single, declarative and modular pipeline  

## vision

deltarecon is not "just a wrapper" of existing tools like nuclei, nikto or ffuf  
it's a **complete operational framework** that:  
- organizes recon and scan tools in ordered and reproducible pipelines,  
- normalizes results in structured formats,  
- uses AI agents (via AutoGen) to filter, correlate and validate findings,  
- produces automatic reports that are immediately ready for validation or submission to bug bounty platforms  

## operational phases

1. **domain**: enumeration of domains, subdomains and attack surfaces
2. **recon**: domain scanning for technology detection
3. **scan**: integration of multiple scanners 
4. **evaluation (AI)**  
   - AI agents via AutoGen
   - findings classification, false positive reduction, pattern correlation  
   - generation of credible and useful pre-reports  

## why it's different

many security tools stop at the stage of "raw" scan execution  
deltarecon instead focuses on three distinctive values:

- **end-to-end automation** → from initial domain to final report, without manual steps  
- **AI evaluation** → faster, targeted and scalable analysis thanks to specialized agents  
- **distributed scalability** → **kubernetes** orchestration, distributed jobs and resilient storage (MinIO for files, Redis for runtime state)  

## in summary

**deltarecon** is much more than a recon framework:  
it's an ecosystem that brings order, intelligence and scalability to a world dominated by isolated tools and manual processes, paving the way for an **automated, intelligent and sustainable** security model

{% include project_sections.html project_name="deltarecon" %}