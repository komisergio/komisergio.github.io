---
title: Integrating Wazuh with Claude via MCP To Enhance Log Analysis — Tutorial & Demonstration (Ubuntu + Wazuh + Claude Desktop + Windows 11)
date: 2025-12-30 05:12 +0000
categories: [Blog]
tags: [blog]
math: true
mermaid: true
---
![1_Ak7ntIHxAqubA1cSrx2-YA](https://github.com/user-attachments/assets/cdd05a60-e98e-4191-9047-f22607d044f8)


Dans le monde en rapide évolution de la cybersécurité, les équipes SOC se retrouvent souvent submergées par une montagne de journaux et d'alertes provenant de leurs systèmes SIEM. Transformer ces données écrasantes en informations utiles prend beaucoup de temps et d'expertise, ce qui peut représenter un véritable défi. Les assistants IA modernes bénéficient grandement d'un contexte réel et structuré. En exposant les données Wazuh via MCP, Claude peut répondre à des requêtes telles que « Quels sont les incidents critiques d'aujourd'hui ? », « Suggérez un playbook pour isoler cette machine » ou « Donnez-moi un tableau de bord exécutif ». Cette intégration rationalise le temps d'enquête, standardise les réponses et améliore la communication entre les équipes SOC et la direction. Le projet mcp-server-wazuh agit comme un pont entre l'API/indexeur Wazuh et les assistants IA via MCP.  ([GitHub](https://github.com/gbrigandi/mcp-server-wazuh?utm_source=chatgpt.com

## Architecture et composants

Dans notre démonstration, l’architecture repose sur :

- **Serveur Ubuntu 24.04** hébergeant les composants Wazuh (manager, indexeur Elasticsearch, dashboard Kibana). Le _Wazuh Manager_ collecte les logs des agents et génère les alertes[documentation.wazuh.com](https://documentation.wazuh.com/current/installation-guide/wazuh-server/step-by-step.html#:~:text=Install%20and%20configure%20the%20Wazuh,events%20to%20the%20Wazuh%20indexer). Le _Wazuh Indexer_ (Elasticsearch) stocke les événements, et le _Wazuh Dashboard_ offre l’interface graphique (Kibana) de visualisation. Le manager expose une API REST sécurisée (par défaut sur le port 55000).
    
- **Agent Wazuh Windows 11** : un agent universel installé sur un poste Windows 11 et inscrit au manager Wazuh. Cet agent récupère les événements locaux (logs, sécurité,…) et les envoie au serveur. Par exemple, on installe l’agent via l’exécutable MSI de Wazuh en ligne de commande (ex. `wazuh-agent-*.msi /q WAZUH_MANAGER="IP_du_manager"`[documentation.wazuh.com](https://documentation.wazuh.com/current/installation-guide/wazuh-agent/wazuh-agent-package-windows.html#:~:text=1,is%20in%20your%20working%20directory)), puis on le démarre (`net start wazuhsvc`). Après inscription, on doit le voir actif dans le dashboard (cf. capture ci-dessous).
    
- **Claude Desktop** : application de bureau Anthropic pour interagir avec le modèle Claude (IA conversationnelle). Nous l’installons sur le même Ubuntu (version Debian/Ubuntu) afin de lancer l’agent MCP localement. Une fois lancé, Claude Desktop propose une interface conviviale pour poser des questions en langage naturel.
    
- **Serveur Wazuh MCP** : serveur de liaison écrit en Rust (projet mcp-server-wazuh). Il agit comme un “traducteur” : il interroge l’API Wazuh et l’indexeur, formate les données et les renvoie à Claude via MCP[atricore.com](https://www.atricore.com/blog/wazuh-mcp-server-bridging-siem-data-with-ai-assistants#:~:text=The%20Wazuh%20MCP%20Server%20acts,understand%20and%20work%20with%20naturally).
    

_Figure : Tableau de bord Wazuh montrant un agent Windows 11 actif (statut “active”). L’agent Windows communique bien avec le serveur Wazuh._

 

Dans l’ensemble, l’agent Windows envoie ses logs au serveur Ubuntu Wazuh. Claude Desktop interroge ensuite ce serveur Ubuntu en local, via le processus MCP, pour obtenir les informations demandées. Le protocole MCP est un standard qui permet à Claude Desktop de lancer un serveur local (notre `mcp-server-wazuh`) et d’échanger des données de façon sécurisée. Cette topologie simplifie les tests en local, mais il est aussi possible de déployer MCP sur un autre hôte ou dans un cluster, si besoin.

## Installation pas à pas

Voici les principales étapes pour mettre en place cet environnement. Nous illustrons chaque étape avec des extraits de code ou captures.

### 1. Installation de Wazuh sur Ubuntu

Sur le serveur Ubuntu 24.04, commencez par configurer le dépôt officiel Wazuh et sa clé GPG, comme indiqué dans la documentation[documentation.wazuh.com](https://documentation.wazuh.com/current/installation-guide/wazuh-server/step-by-step.html#:~:text=2). Par exemple :

Téléchargez et exécutez l'assistant d'installation de Wazuh.
````shell
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh && sudo bash ./wazuh-install.sh -a
`````

Une fois l'assistant terminé l'installation, la sortie affiche les informations d'accès et un message qui confirme que l'installation a été réussie.

````
INFO: --- Summary ---
INFO: You can access the web interface https://<WAZUH_DASHBOARD_IP_ADDRESS>
    User: admin
    Password: <ADMIN_PASSWORD>
INFO: Installation finished.
```````

You now have installed and configured Wazuh.

2. Access the Wazuh web interface with `https://<WAZUH_DASHBOARD_IP_ADDRESS>` and your credentials:
    
    - **Username**: `admin`
        
    - **Password**: `<ADMIN_PASSWORD>`
        

When you access the Wazuh dashboard for the first time, the browser shows a warning message stating that the certificate was not issued by a trusted authority. This is expected and the user has the option to accept the certificate as an exception or, alternatively, configure the system to use a certificate from a trusted authority.

Note

You can find the passwords for all the Wazuh indexer and Wazuh API users in the `wazuh-passwords.txt` file inside `wazuh-install-files.tar`. To print them, run the following command:

````shell
sudo tar -O -xvf wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt
```````

**Recommended Action**: Disable Wazuh Updates.

We recommend disabling the Wazuh package repositories after installation to prevent accidental upgrades that could break the environment.

Execute the following command to disable the Wazuh repository:

````shell
sed -i "s/^deb /#deb /" /etc/apt/sources.list.d/wazuh.list

apt update
``````


_Figure : Terminal Ubuntu après un script d’installation de Wazuh. Le script installe le manager, Elasticsearch, Filebeat et le dashboard, puis indique les URL et identifiants d’accès (admin:admin par défaut)._

 

Cette capture illustre un script d’installation automatisé de Wazuh sur Ubuntu. En sortie, vous devriez pouvoir accéder à l’interface web du dashboard (par exemple https://IP_Wazuh:443 avec l’utilisateur `admin`). 

### 2. Installation de Claude Desktop

Pour Claude Desktop sur Ubuntu/Debian, le plus simple est de télécharger le package .deb depuis la page GitHub du projet claude-desktop-debian. Par exemple, téléchargez la dernière release :

````shell
wget https://github.com/aaddrick/claude-desktop-debian/releases/download/vX.Y.Z/claude-desktop_X.Y.Z_amd64.deb
sudo dpkg -i claude-desktop_X.Y.Z_amd64.deb
sudo apt --fix-broken install  # pour installer les dépendances manquantes
`````

La doc confirme cette procédure avec `dpkg -i` (voir extrait ci-dessous)[github.com](https://github.com/aaddrick/claude-desktop-debian?tab=readme-ov-file#:~:text=For%20) :
For .deb packages:
````shell
    #sudo dpkg -i ./claude-desktop_VERSION_ARCHITECTURE.deb 
    sudo dpkg -i ./*.deb
    sudo apt --fix-broken install
```````

Une fois installé, lancez **Claude Desktop** (depuis le menu ou la commande `claude`). L’interface graphique apparaîtra. On voit ci-dessous Claude Desktop installé sur Ubuntu (icône « Claude Desktop » à gauche).

 

_Figure : Claude Desktop installé sur Ubuntu (icône en bas). L’interface de Claude est prête à être configurée._

### 3. Installation et configuration du serveur MCP

Le serveur MCP pour Wazuh (`mcp-server-wazuh`) peut être installé soit en compilant le projet Rust, soit en utilisant un binaire fourni. Voici la méthode Rust (comme recommandé)[augmentcode.com](https://www.augmentcode.com/mcp/mcp-server-wazuh#:~:text=2) :

````shell
# Installer Rust s'il n'est pas déjà présent (rustup)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
# Ensuite :
git clone https://github.com/gbrigandi/mcp-server-wazuh.git
cd mcp-server-wazuh
cargo build --release
```````

Le binaire compilé se trouve alors dans `target/release/mcp-server-wazuh`. (Une autre option est de récupérer directement une release précompilée depuis GitHub, ou de déployer via Docker si disponible.)

#### Configuration du fichier `claude_desktop_config.json`

Il faut maintenant indiquer à Claude Desktop comment lancer ce serveur MCP et avec quelles options. Dans **Claude Desktop > Fichier > Paramètres > Développeur**, cliquez sur **Edit Config**. Le fichier de configuration MCP se situe par défaut dans `~/.config/Claude/claude_desktop_config.json`[github.com](https://github.com/aaddrick/claude-desktop-debian?tab=readme-ov-file#:~:text=Model%20Context%20Protocol%20settings%20are,stored%20in). Modifiez-le (ou créez-le) pour y inclure un bloc `mcpServers` pour Wazuh. Par exemple :

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
        "WAZUH_API_PASSWORD": "votre_mot_de_passe",
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


Figure config.json

Dans cet exemple on indique le chemin du binaire MCP et on définit les variables d’environnement nécessaires (hôte, ports, utilisateurs/mots de passe pour l’API et l’indexeur Wazuh)[augmentcode.com](https://www.augmentcode.com/mcp/mcp-server-wazuh#:~:text=1). Enregistrez ce fichier et redémarrez Claude Desktop.

#### Vérification du lien Claude–Wazuh

Retournez dans **Claude Desktop > Développeur**. Vous devriez voir un statut « **wazuh running** » indiquant que le serveur MCP s’est lancé correctement (comme sur la capture ci-dessous). Ensuite, dans l’onglet **Connecteurs** du menu principal, activez l’interrupteur **Wazuh** pour autoriser la connexion.

 

_Figure : Après configuration, Claude Desktop affiche “wazuh running” dans les paramètres Développeur et le connecteur Wazuh est activé. Le pont MCP est en ligne._

 

Pour tester, posez une question à Claude sur Wazuh. Par exemple : « Combien d’agents Wazuh avez-vous et combien d’alertes critiques sont présentes ? ». Claude va invoquer la fonction MCP correspondante (p. ex. `get_wazuh_alert_summary`) et rendre le résultat. S’il répond de manière cohérente (en résumé il devrait lister le nombre d’agents actifs et d’alertes critiques détectées), alors l’intégration fonctionne.

## Cas d’usage illustrés

prompt 1 :

````prompt
As you access my Wazuh SIEM environment, act as a security analyst. 
- Analyze logs, alerts, and events with a focus on detecting threats, anomalies, and compliance issues.  
- Provide clear explanations of findings, including severity, potential impact, and recommended actions.  
- Suggest improvements to security posture and SIEM configuration when relevant.  
- Always structure responses in a concise, actionable format (summary + detailed insights + recommendations).  
```````

les capture d écran sont jointes. 


prompt 2 : 

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

les capture d écran sont jointes 

## Sécurité et bonnes pratiques

Intégrer un LLM à votre SOC implique certaines précautions :

- **Sanitisation des données** : Ne pas envoyer au LLM d’informations sensibles non anonymisées (ex. mots de passe, clés privées, données personnelles, adresses IP internes), car Claude est un service externe qui pourrait les stocker temporairement. Filtrez ou anonymisez les logs avant de les exposer. Évitez par exemple de demander directement les logs bruts.
    
- **Validation humaine** : Les suggestions de l’IA sont à prendre en assistant, pas à appliquer aveuglément. Toujours faire valider manuellement les détections ou les actions générées (playbooks, règles, etc.). Les LLM peuvent halluciner ou généraliser, il faut vérifier notamment qu’une nouvelle règle est correcte et non trop permissive.
    
- **Audit et traçabilité** : Conservez un historique des prompts envoyés à Claude et des réponses reçues. Cela permet de vérifier pourquoi une décision a été suggérée et d’auditer le système en cas d’erreur. Intégrez si possible cette piste d’audit dans vos journaux SIEM.
    
- **Contrôle d’accès** : Limitez qui dans l’équipe peut utiliser cette intégration. Par exemple, réservez-la aux analystes seniors et ajustez les permissions de l’outil Claude Desktop selon la politique interne.
    
- **Gouvernance IA** : Mettez en place une revue régulière de l’utilisation de Claude, par exemple des checklists d’usage ou des revues post-mortem d’alertes traitées par l’IA. Entraînez les utilisateurs à reconnaître les limites des modèles et à ne pas les laisser effectuer d’actions sensibles automatiquement.
    

En résumé, l’IA doit rester un _assistant_ qui améliore la productivité, et non un « pilote » autonome du SOC.

## Conclusion et perspectives

Cet article a montré comment associer **Wazuh** et **Claude Desktop** via le serveur **mcp-server-wazuh** pour enrichir vos analyses. Cette intégration illustre un changement de paradigme en SOC : passer du **data extraction** à l’**analysis en langage naturel**[atricore.com](https://www.atricore.com/blog/wazuh-mcp-server-bridging-siem-data-with-ai-assistants#:~:text=Traditional%20SIEM%20workflows%20require%20specialized,a%20short%20period%20of%20time). Le lien Claude–Wazuh ouvre la porte à de nombreuses extensions possibles. Par exemple, on peut imaginer envoyer les résultats directement vers un système de ticketing (TheHive) ou déclencher des playbooks automatisés via Cortex. On pourrait aussi enrichir les requêtes avec des données de MISP (OSINT) pour contextualiser les alertes.


En guise d’appels à l’action : explorez d’autres assistants IA (ex. ChatGPT avec plugin, ou Llama via Ollama) avec Wazuh, partagez vos scripts et cas d’usage dans la communauté, et testez de nouvelles idées (comme l’analyse sémantique des logs via embeddings). À terme, l’objectif est d’intégrer l’IA de manière responsable dans le workflow SOC (playbooks dynamiques, rapports automatisés, détection avancée). Mais n’oublions pas : l’expertise humaine reste centrale. Claude Desktop est un outil puissant, à utiliser avec discernement et suivi.

 

**Sources :** 

Documentation Wazuh pour l’installation et la configuration[documentation.wazuh.com](https://documentation.wazuh.com/current/installation-guide/wazuh-server/step-by-step.html#:~:text=2)[documentation.wazuh.com](https://documentation.wazuh.com/current/installation-guide/wazuh-agent/wazuh-agent-package-windows.html#:~:text=1,is%20in%20your%20working%20directory) ; blog technique Atricore sur le MCP Wazuh[atricore.com](https://www.atricore.com/blog/wazuh-mcp-server-bridging-siem-data-with-ai-assistants#:~:text=The%20Wazuh%20MCP%20Server%20addresses,data%20and%20automated%20analysis%20workflows)[atricore.com](https://www.atricore.com/blog/wazuh-mcp-server-bridging-siem-data-with-ai-assistants#:~:text=The%20Wazuh%20MCP%20Server%20acts,understand%20and%20work%20with%20naturally)[atricore.com](https://www.atricore.com/blog/wazuh-mcp-server-bridging-siem-data-with-ai-assistants#:~:text=Automated%20alert%20triage%3A%20AI%20assistants,reviewing%20hundreds%20of%20alerts%20daily)[atricore.com](https://www.atricore.com/blog/wazuh-mcp-server-bridging-siem-data-with-ai-assistants#:~:text=Traditional%20SIEM%20workflows%20require%20specialized,a%20short%20period%20of%20time) ; dépôts GitHub pour Claude Desktop[github.com](https://github.com/aaddrick/claude-desktop-debian?tab=readme-ov-file#:~:text=For%20) et Wazuh MCP[augmentcode.com](https://www.augmentcode.com/mcp/mcp-server-wazuh#:~:text=2)[augmentcode.com](https://www.augmentcode.com/mcp/mcp-server-wazuh#:~:text=1).

Citations

Wazuh MCP server: Bridging SIEM data with AI assistants

https://www.atricore.com/blog/wazuh-mcp-server-bridging-siem-data-with-ai-assistants

](https://www.atricore.com/blog/wazuh-mcp-server-bridging-siem-data-with-ai-assistants#:~:text=Security%20operations%20teams%20face%20a,overwhelm%20you%20and%20your%20team)[



Wazuh MCP server: Bridging SIEM data with AI assistants

https://www.atricore.com/blog/wazuh-mcp-server-bridging-siem-data-with-ai-assistants

](https://www.atricore.com/blog/wazuh-mcp-server-bridging-siem-data-with-ai-assistants#:~:text=The%20Wazuh%20MCP%20Server%20addresses,data%20and%20automated%20analysis%20workflows)[



Wazuh MCP server: Bridging SIEM data with AI assistants

https://www.atricore.com/blog/wazuh-mcp-server-bridging-siem-data-with-ai-assistants

](https://www.atricore.com/blog/wazuh-mcp-server-bridging-siem-data-with-ai-assistants#:~:text=The%20Wazuh%20MCP%20Server%20acts,understand%20and%20work%20with%20naturally)[


Wazuh MCP server: Bridging SIEM data with AI assistants

https://www.atricore.com/blog/wazuh-mcp-server-bridging-siem-data-with-ai-assistants

](https://www.atricore.com/blog/wazuh-mcp-server-bridging-siem-data-with-ai-assistants#:~:text=Automated%20alert%20triage%3A%20AI%20assistants,reviewing%20hundreds%20of%20alerts%20daily)[

Installing the Wazuh server step by step - Wazuh server

https://documentation.wazuh.com/current/installation-guide/wazuh-server/step-by-step.html

](https://documentation.wazuh.com/current/installation-guide/wazuh-server/step-by-step.html#:~:text=Install%20and%20configure%20the%20Wazuh,events%20to%20the%20Wazuh%20indexer)[


Deploying Wazuh agents on Windows endpoints - Wazuh agent

https://documentation.wazuh.com/current/installation-guide/wazuh-agent/wazuh-agent-package-windows.html

](https://documentation.wazuh.com/current/installation-guide/wazuh-agent/wazuh-agent-package-windows.html#:~:text=1,is%20in%20your%20working%20directory)[

Installing the Wazuh server step by step - Wazuh server

https://documentation.wazuh.com/current/installation-guide/wazuh-server/step-by-step.html

](https://documentation.wazuh.com/current/installation-guide/wazuh-server/step-by-step.html#:~:text=2)[

Installing the Wazuh server step by step - Wazuh server

https://documentation.wazuh.com/current/installation-guide/wazuh-server/step-by-step.html

](https://documentation.wazuh.com/current/installation-guide/wazuh-server/step-by-step.html#:~:text=1,package)[

Installing the Wazuh server step by step - Wazuh server

https://documentation.wazuh.com/current/installation-guide/wazuh-server/step-by-step.html

](https://documentation.wazuh.com/current/installation-guide/wazuh-server/step-by-step.html#:~:text=APTYumDNF)[


Installing the Wazuh server step by step - Wazuh server

https://documentation.wazuh.com/current/installation-guide/wazuh-server/step-by-step.html

](https://documentation.wazuh.com/current/installation-guide/wazuh-server/step-by-step.html#:~:text=1.%20,Wazuh%20indexer%20IP%20address%20accordingly)[

GitHub - aaddrick/claude-desktop-debian: Claude Desktop for Debian-based Linux distributions

https://github.com/aaddrick/claude-desktop-debian?tab=readme-ov-file

](https://github.com/aaddrick/claude-desktop-debian?tab=readme-ov-file#:~:text=For%20)[

mcp-server-wazuh - MCP Server Registry - Augment Code

https://www.augmentcode.com/mcp/mcp-server-wazuh

](https://www.augmentcode.com/mcp/mcp-server-wazuh#:~:text=2)[


GitHub - aaddrick/claude-desktop-debian: Claude Desktop for Debian-based Linux distributions

https://github.com/aaddrick/claude-desktop-debian?tab=readme-ov-file

](https://github.com/aaddrick/claude-desktop-debian?tab=readme-ov-file#:~:text=Model%20Context%20Protocol%20settings%20are,stored%20in)[

)

mcp-server-wazuh - MCP Server Registry - Augment Code

https://www.augmentcode.com/mcp/mcp-server-wazuh

](https://www.augmentcode.com/mcp/mcp-server-wazuh#:~:text=1)[

Wazuh MCP server: Bridging SIEM data with AI assistants

https://www.atricore.com/blog/wazuh-mcp-server-bridging-siem-data-with-ai-assistants

](https://www.atricore.com/blog/wazuh-mcp-server-bridging-siem-data-with-ai-assistants#:~:text=Traditional%20SIEM%20workflows%20require%20specialized,a%20short%20period%20of%20time)

Toutes les sources
