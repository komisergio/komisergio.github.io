---
title: Integrating Wazuh with Claude via MCP To Enhance Log Analysis — Tutorial & Demonstration (Ubuntu + Wazuh + Claude Desktop + Windows 11)
date: 2025-12-30 05:12 +0000
categories: [Blog]
tags: [blog]
math: true
mermaid: true
---

In the fast-evolving world of cybersecurity, SOC teams often find themselves buried under a mountain of logs and alerts from their SIEM systems. Turning this overwhelming data into useful insights takes a lot of time and expertise, which can be a real challenge. Modern AI assistants greatly benefit from being provided with real and structured context. By exposing Wazuh data through MCP, Claude can respond to queries such as “What are today’s critical incidents?”, “Suggest a playbook to isolate this machine,” or “Give me an executive dashboard.” This integration streamlines investigation time, standardizes responses, and enhances communication between SOC teams and management. The mcp-server-wazuh project acts as a bridge between the Wazuh API/indexer and AI assistants via MCP.  ([GitHub](https://github.com/gbrigandi/mcp-server-wazuh?utm_source=chatgpt.com "MCP Server for Wazuh SIEM - GitHub"))


