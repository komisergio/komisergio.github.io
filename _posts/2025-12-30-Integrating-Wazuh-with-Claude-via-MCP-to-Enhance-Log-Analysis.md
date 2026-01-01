---
title: Integrating Wazuh with Claude via MCP To Enhance Log Analysis
date: 2025-12-30 05:12 +0000
categories: [Blog]
tags: [blog]
math: true
mermaid: true
---
![1_Ak7ntIHxAqubA1cSrx2-YA](https://github.com/user-attachments/assets/cdd05a60-e98e-4191-9047-f22607d044f8)

## Integrating Wazuh with Claude via MCP To Enhance Log Analysis — Tutorial & Demonstration (Ubuntu + Wazuh + Claude Desktop + Windows 11)

In the fast-evolving world of cybersecurity, SOC teams often find themselves buried under a mountain of logs and alerts from their SIEM systems. Turning this overwhelming data into useful insights takes a lot of time and expertise, which can be a real challenge. Modern AI assistants greatly benefit from being provided with real and structured context. By exposing Wazuh data through MCP, Claude can respond to queries such as “What are today’s critical incidents?”, “Suggest a playbook to isolate this machine,” or “Give me an executive dashboard.” This integration streamlines investigation time, standardizes responses, and enhances communication between SOC teams and management. The mcp-server-wazuh project acts as a bridge between the Wazuh API/indexer and AI assistants via MCP.  ([GitHub](https://github.com/gbrigandi/mcp-server-wazuh?utm_source=chatgpt.com "MCP Server for Wazuh SIEM - GitHub"))

## Architecture and Components

In our demonstration, the architecture is based on:

- **Ubuntu 24.04 Server** hosting Wazuh components (manager, Elasticsearch indexer, Kibana dashboard). The _Wazuh Manager_ collects logs from agents and generates alerts [documentation.wazuh.com](https://documentation.wazuh.com/current/installation-guide/wazuh-server/step-by-step.html#:~:text=Install%20and%20configure%20the%20Wazuh,events%20to%20the%20Wazuh%20indexer). The _Wazuh Indexer_ (Elasticsearch) stores events, and the _Wazuh Dashboard_ provides the graphical interface (Kibana) for visualization. The manager exposes a secure REST API (by default on port 55000).

<img width="1919" height="917" alt="Ubuntu Interface" src="https://github.com/user-attachments/assets/5c9ad156-79e8-47bc-a738-3368ec339263" />

- **Wazuh Agent Windows 11**: a universal agent installed on a Windows 11 workstation and registered with the Wazuh manager. This agent retrieves local events (logs, security, etc.) and sends them to the server. For example, the agent is installed via the Wazuh MSI executable from the command line (e.g., `wazuh-agent-*.msi /q WAZUH_MANAGER="IP_of_manager"` [documentation.wazuh.com](https://documentation.wazuh.com/current/installation-guide/wazuh-agent/wazuh-agent-package-windows.html#:~:text=1,is%20in%20your%20working%20directory)), and then started (`net start wazuhsvc`). After registration, it should be visible as active in the dashboard

<img width="1919" height="919" alt="Windows 11 agnet wazuh" src="https://github.com/user-attachments/assets/f33080e2-688a-4dec-a7a1-d11207c1130e" />

- **Claude Desktop**: Anthropic's desktop application for interacting with the Claude model (conversational AI). We install it on the same Ubuntu (Debian/Ubuntu version) to run the MCP agent locally. Once launched, Claude Desktop offers a user-friendly interface for asking questions in natural language.

<img width="1917" height="915" alt="claude desktop" src="https://github.com/user-attachments/assets/434e6ec9-e4d8-460b-8c61-a84dc49e4b56" />


- **Wazuh MCP Server**: a bridging server written in Rust (project mcp-server-wazuh). It acts as a “translator”: it queries the Wazuh API and indexer, formats the data, and sends it back to Claude via MCP [atricore.com](https://www.atricore.com/blog/wazuh-mcp-server-bridging-siem-data-with-ai-assistants#:~:text=The%20Wazuh%20MCP%20Server%20acts,understand%20and%20work%20with%20naturally).

<img width="1919" height="952" alt="dashbord wazuh" src="https://github.com/user-attachments/assets/83db27a6-1077-40ae-98dd-da6253734223" />

Overall, the Windows agent sends its logs to the Wazuh Ubuntu server. Claude Desktop then queries this Ubuntu server locally, via the MCP process, to obtain the requested information. The MCP protocol is a standard that allows Claude Desktop to launch a local server (our `mcp-server-wazuh`) and exchange data securely. This topology simplifies local testing, but it is also possible to deploy MCP on another host or in a cluster if needed.

## Step-by-Step Installation

Here are the main steps to set up this environment. We illustrate each step with code snippets or screenshots.

### 1. Installing Wazuh on Ubuntu

On the Ubuntu 24.04 server, start by configuring the official Wazuh repository and its GPG key, as indicated in the documentation [documentation.wazuh.com](https://documentation.wazuh.com/current/installation-guide/wazuh-server/step-by-step.html#:~:text=2). For example:

Download and run the Wazuh installation script.

````shell
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh && sudo bash ./wazuh-install.sh -a
````

Once the installation script is finished, the output displays access information and a message confirming that the installation was successful.

````
INFO: --- Summary ---
INFO: You can access the web interface https://<WAZUH_DASHBOARD_IP_ADDRESS>
    User: admin
    Password: <ADMIN_PASSWORD>
INFO: Installation finished.
``````
You now have installed and configured Wazuh.

2. Access the Wazuh web interface with `https://<WAZUH_DASHBOARD_IP_ADDRESS>` and your credentials:
    
    - **Username**: `admin`
        
    - **Password**: `<ADMIN_PASSWORD>`
        

When you access the Wazuh dashboard for the first time, the browser shows a warning message stating that the certificate was not issued by a trusted authority. This is expected, and the user has the option to accept the certificate as an exception or, alternatively, configure the system to use a certificate from a trusted authority.

Note

You can find the passwords for all the Wazuh indexer and Wazuh API users in the `wazuh-passwords.txt` file inside `wazuh-install-files.tar`. To print them, run the following command:

````shell
sudo tar -O -xvf wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt
``````

**Recommended Action**: Disable Wazuh Updates.

We recommend disabling the Wazuh package repositories after installation to prevent accidental upgrades that could break the environment.

Execute the following command to disable the Wazuh repository:

````shell
sed -i "s/^deb /#deb /" /etc/apt/sources.list.d/wazuh.list

apt update
``````
<img width="1917" height="922" alt="Wazuh Installation avec le script sh" src="https://github.com/user-attachments/assets/f47a3222-6001-46e3-b072-ea6c0b4e60a7" />

This screenshot illustrates an automated Wazuh installation script on Ubuntu. In output, you should be able to access the web interface of the dashboard (e.g., https://IP_Wazuh:443 with the user `admin`).

### 2. Installing Claude Desktop

For Claude Desktop on Ubuntu/Debian, the simplest method is to download the .deb package from the GitHub page of the claude-desktop-debian project. For example, download the latest release:

````shell
wget https://github.com/aaddrick/claude-desktop-debian/releases/download/vX.Y.Z/claude-desktop_X.Y.Z_amd64.deb
sudo dpkg -i claude-desktop_X.Y.Z_amd64.deb
sudo apt --fix-broken install  # to install missing dependencies
````

The documentation confirms this procedure with `dpkg -i` (see excerpt below) [github.com](https://github.com/aaddrick/claude-desktop-debian?tab=readme-ov-file#:~:text=For%20):
For .deb packages:
````shell
    #sudo dpkg -i ./claude-desktop_VERSION_ARCHITECTURE.deb 
    sudo dpkg -i ./*.deb
    sudo apt --fix-broken install
``````

Once installed, launch **Claude Desktop** (from the menu or the command `claude`). The graphical interface will appear. Below, we see Claude Desktop installed on Ubuntu (the "Claude Desktop" icon on the left).

<img width="1919" height="942" alt="Claude Desktop Instale" src="https://github.com/user-attachments/assets/db09649b-accd-4653-b4eb-bb5355e6b114" />



### 3. Installing and Configuring the MCP Server

The MCP server for Wazuh (`mcp-server-wazuh`) can be installed either by compiling the Rust project or by using a provided binary. Here is the Rust method (as recommended) [augmentcode.com](https://www.augmentcode.com/mcp/mcp-server-wazuh#:~:text=2):

````shell
# Install Rust if not already present (rustup)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
# Then:
git clone https://github.com/gbrigandi/mcp-server-wazuh.git
cd mcp-server-wazuh
cargo build --release
``````

The compiled binary can then be found in `target/release/mcp-server-wazuh`. (Another option is to directly obtain a precompiled release from GitHub or deploy via Docker if available.)

#### Configuring the `claude_desktop_config.json` File

Now you need to tell Claude Desktop how to launch this MCP server and with what options. In **Claude Desktop > File > Settings > Developer**, click on **Edit Config**. The MCP configuration file is located by default in `~/.config/Claude/claude_desktop_config.json` [github.com](https://github.com/aaddrick/claude-desktop-debian?tab=readme-ov-file#:~:text=Model%20Context%20Protocol%20settings%20are,stored%20in). Modify it (or create it) to include an `mcpServers` block for Wazuh. For example:

````json
{
  "mcpServers": {
    "wazuh": {
      "command": "/home/ubuntu/mcp-server-wazuh/target/release/mcp-server-wazuh",
      "args": [],
      "env": {
        "WAZUH_API_HOST": "localhost",
        "WAZUH_API_PORT": "55000",
        "WAZUH_API_USERNAME": "wazuh",
        "WAZUH_API_PASSWORD": "your_password",
        "WAZUH_INDEXER_HOST": "localhost",
        "WAZUH_INDEXER_PORT": "9200",
        "WAZUH_INDEXER_USERNAME": "admin",
        "WAZUH_INDEXER_PASSWORD": "admin",
        "WAZUH_VERIFY_SSL": "false",
        "WAZUH_TEST_PROTOCOL": "https",
        "RUST_LOG": "info"
      }
    }
  }
}
`````

_Figure: config.json_

In this example, we specify the path to the MCP binary and define the necessary environment variables (host, ports, users/passwords for the Wazuh API and indexer) [augmentcode.com](https://www.augmentcode.com/mcp/mcp-server-wazuh#:~:text=1). Save this file and restart Claude Desktop.

#### Verifying the Claude-Wazuh Link

Go back to **Claude Desktop > Developer**. You should see a status “**wazuh running**” indicating that the MCP server has started correctly (as in the screenshot below). Then, in the **Connectors** tab of the main menu, toggle the **Wazuh** switch to allow the connection.

_Figure: After configuration, Claude Desktop displays “wazuh running” in the Developer settings, and the Wazuh connector is enabled. The MCP bridge is online._

To test, ask Claude a question about Wazuh. For example: “How many Wazuh agents do you have and how many critical alerts are present?” Claude will invoke the corresponding MCP function (e.g., `get_wazuh_alert_summary`) and return the result. If it responds coherently (in summary, it should list the number of active agents and critical alerts detected), then the integration works.

## Illustrated Use Cases

prompt 1:

````prompt
As you access my Wazuh SIEM environment, act as a security analyst. 
- Analyze logs, alerts, and events with a focus on detecting threats, anomalies, and compliance issues.  
- Provide clear explanations of findings, including severity, potential impact, and recommended actions.  
- Suggest improvements to security posture and SIEM configuration when relevant.  
- Always structure responses in a concise, actionable format (summary + detailed insights + recommendations).  
```````

Screenshots are attached.

prompt 2:

````prompt
dashboard.
Create an executive-level security dashboard using Wazuh SIEM data from the Windows 11 workstation.  
The dashboard should present information in simple, non-technical language with easy-to-understand indicators.  
Focus on the following elements:

- Overall risk level (use clear categories such as Low / Medium / High)  
- Main threats identified (summarize in plain terms, avoid jargon)  
- Incident trends (show whether threats are increasing, stable, or decreasing)  
- Potential impacts (on data, system availability, and compliance)  
- Recommended short-term actions (practical steps, expressed simply)  

Structure the output as a concise, executive summary with visual-style indicators (e.g., traffic light colors, arrows, or simple icons).
```````

Screenshots are attached.

## Security and Best Practices

Integrating a LLM into your SOC involves certain precautions:

- **Data Sanitization**: Do not send sensitive non-anonymized information to the LLM (e.g., passwords, private keys, personal data, internal IP addresses), as Claude is an external service that could temporarily store them. Filter or anonymize logs before exposing them. For example, avoid directly requesting raw logs.

- **Human Validation**: AI suggestions should be taken as assistance, not applied blindly. Always manually validate detections or actions generated (playbooks, rules, etc.). LLMs can hallucinate or generalize; it's crucial to check that a new rule is correct and not too permissive.

- **Audit and Traceability**: Keep a history of the prompts sent to Claude and the responses received. This allows you to verify why a decision was suggested and to audit the system in case of an error. Integrate this audit trail into your SIEM logs if possible.

- **Access Control**: Limit who in the team can use this integration. For example, reserve it for senior analysts and adjust the permissions of the Claude Desktop tool according to internal policy.

- **AI Governance**: Implement a regular review of Claude's usage, such as usage checklists or post-mortem reviews of alerts processed by AI. Train users to recognize the limitations of models and not to let them perform sensitive actions automatically.

In summary, AI should remain an _assistant_ that enhances productivity, not an autonomous “pilot” of the SOC.

## Conclusion and Perspectives

This article has shown how to associate **Wazuh** and **Claude Desktop** via the **mcp-server-wazuh** server to enrich your analyses. This integration illustrates a paradigm shift in SOC: moving from **data extraction** to **natural language analysis** [atricore.com](https://www.atricore.com/blog/wazuh-mcp-server-bridging-siem-data-with-ai-assistants#:~:text=Traditional%20SIEM%20workflows%20require%20specialized,a%20short%20period%20of%20time). The Claude-Wazuh link opens the door to many possible extensions. For example, one could imagine sending results directly to a ticketing system (TheHive) or triggering automated playbooks via Cortex. We could also enrich queries with MISP (OSINT) data to contextualize alerts.

As calls to action: explore other AI assistants (e.g., ChatGPT with plugins, or Llama via Ollama) with Wazuh, share your scripts and use cases within the community, and test new ideas (like semantic log analysis via embeddings). Ultimately, the goal is to integrate AI responsibly into the SOC workflow (dynamic playbooks, automated reports, advanced detection). But let’s not forget: human expertise remains central. Claude Desktop is a powerful tool, to be used judiciously and monitored.

**Sources:**

Wazuh documentation for installation and configuration [documentation.wazuh.com](https://documentation.wazuh.com/current/installation-guide/wazuh-server/step-by-step.html#:~:text=2) [documentation.wazuh.com](https://documentation.wazuh.com/current/installation-guide/wazuh-agent/wazuh-agent-package-windows.html#:~:text=1,is%20in%20your%20working%20directory); Atricore technical blog on Wazuh MCP [atricore.com](https://www.atricore.com/blog/wazuh-mcp-server-bridging-siem-data-with-ai-assistants#:~:text=The%20Wazuh%20MCP%20Server%20addresses,data%20and%20automated%20analysis%20workflows) [atricore.com](https://www.atricore.com/blog/wazuh-mcp-server-bridging-siem-data-with-ai-assistants#:~:text=The%20Wazuh%20MCP%20Server%20acts,understand%20and%20work%20with%20naturally) [atricore.com](https://www.atricore.com/blog/wazuh-mcp-server-bridging-siem-data-with-ai-assistants#:~:text=Automated%20alert%20triage%3A%20AI%20assistants,reviewing%20hundreds%20of%20alerts%20daily) [atricore.com](https://www.atricore.com/blog/wazuh-mcp-server-bridging-siem-data-with-ai-assistants#:~:text=Traditional%20SIEM%20workflows%20require%20specialized,a%20short%20period%20of%20time); GitHub repositories for Claude Desktop [github.com](https://github.com/aaddrick/claude-desktop-debian?tab=readme-ov-file#:~:text=For%20) and Wazuh MCP [augmentcode.com](https://www.augmentcode.com/mcp/mcp-server-wazuh#:~:text=2) [augmentcode.com](https://www.augmentcode.com/mcp/mcp-server-wazuh#:~:text=1).

Citations

Wazuh MCP server: Bridging SIEM data with AI assistants

https://www.atricore.com/blog/wazuh-mcp-server-bridging-siem-data-with-ai-assistants
