# AppConnectionFinder

**Français — English below : [version anglaise](#english).**

Identifiez les applications Windows qui communiquent avec un nom d'hôte ou une adresse IP, à partir d'une interface graphique. AppConnectionFinder combine les connexions TCP actives, les événements de connexion Windows et les requêtes DNS pour faciliter le diagnostic réseau.

L'interface de l'application est en anglais.

![Interface d'AppConnectionFinder](screenshot_ui.png)

## Fonctionnalités

- Surveillance ciblée d'une ou plusieurs URL, noms d'hôte ou adresses IPv4/IPv6.
- Résolution des noms d'hôte et actualisation périodique des adresses IP.
- Identification des processus : nom, PID, chemin de l'exécutable et société, lorsque disponibles.
- Corrélation des connexions TCP, des flux UDP et des requêtes DNS.
- Mode **FULL capture**, sans cible, pour observer l'activité de toutes les applications.
- Vérification ponctuelle des connexions TCP avec **Check now (snapshot)**.
- Intervalle de surveillance réglable de 1 à 30 secondes, avec une valeur par défaut de 2 secondes.
- Affichage des premières et dernières observations, d'un compteur et des détails de connexion.
- Journalisation CSV automatique pendant la surveillance.

## Prérequis

- Windows avec une session graphique : l'interface utilise Windows Forms.
- Pour le script : Windows PowerShell 5.1 ou PowerShell 7+ sous Windows.
- Droits administrateur pour les fonctionnalités complètes, notamment l'audit des connexions, la corrélation DNS et certains détails des processus.
- Droit d'écriture dans le dossier de l'application pour créer les journaux.

Le script utilise les composants intégrés à Windows et ne nécessite pas de module PowerShell tiers.

## Démarrage

### Exécutable

Lancez [`AppConnectionFinder.exe`](AppConnectionFinder.exe) depuis un dossier dans lequel vous pouvez écrire, puis acceptez la demande d'élévation UAC pour disposer de toutes les fonctionnalités.

### Script PowerShell

Depuis le dossier du projet :

```powershell
# Windows PowerShell 5.1
powershell.exe -NoProfile -STA -File .\AppConnectionFinder.ps1

# PowerShell 7+
pwsh.exe -NoProfile -STA -File .\AppConnectionFinder.ps1
```

Si la stratégie d'exécution bloque le script et que les règles de votre organisation l'autorisent, vous pouvez utiliser une exception limitée au processus lancé :

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass -STA -File .\AppConnectionFinder.ps1
```

L'application tente de se relancer avec élévation si nécessaire. En cas de refus ou d'échec, elle reste accessible avec des fonctionnalités réduites ; consultez **Activity log** pour les avertissements.

## Utilisation

### Surveiller une cible

1. Saisissez une URL, un nom d'hôte ou une adresse IP dans **URL / hostname or IP address**.
2. Cliquez sur **Add target**. Vous pouvez ajouter plusieurs cibles séparées par des virgules.
3. Cochez les cibles à surveiller et choisissez **Poll interval (seconds)**.
4. Cliquez sur **Start logging**, puis reproduisez l'activité réseau à diagnostiquer.
5. Consultez les processus et les méthodes de détection dans le tableau.
6. Cliquez sur **Stop logging**, puis sur **Open log folder** pour retrouver le CSV.

Exemples de saisie :

```text
https://www.microsoft.com/en-us
example.com
192.0.2.10
2001:db8::10
example.com, 192.0.2.10
```

**Important :** une URL est ramenée à son nom d'hôte ou à son adresse IP. Le schéma, le chemin et le port saisis ne constituent pas des filtres : ce n'est pas un outil de suivi d'une URL HTTP précise.

### Observer toutes les applications

Cliquez sur **Start FULL capture (all connections, no filter)** et confirmez. Aucune cible n'est nécessaire. Ce mode collecte les connexions TCP visibles, y compris les sockets en écoute, les événements de flux UDP et les requêtes DNS disponibles, sans filtre de cible.

Sur une machine active, ce mode peut générer beaucoup de données. Limitez la durée à celle nécessaire au diagnostic.

### Autres commandes

- **Check now (snapshot)** : vérifie uniquement les connexions TCP actives correspondant aux cibles cochées. Pour l'UDP et les événements DNS, utilisez la surveillance continue.
- **Clear results** : efface le tableau, sans supprimer ni vider le fichier CSV existant.
- **Remove selected / Clear all** : retire les cibles sélectionnées ou toutes les cibles.

## Méthodes de détection et limites

| Méthode | Source | Interprétation |
|---|---|---|
| TCP | `Get-NetTCPConnection`, avec repli sur `netstat` | Associe l'adresse distante aux IP de la cible et identifie le processus propriétaire. Une connexion très brève peut échapper à l'échantillonnage. |
| UDP | Journal Security, événement Windows Filtering Platform 5156 | Exploite l'audit des connexions autorisées pour identifier le processus et les extrémités du flux. Nécessite l'accès au journal et à la stratégie d'audit. |
| DNS | `Microsoft-Windows-DNS-Client/Operational`, événements 3006/3008 | Associe une requête DNS au processus indiqué par Windows. Une requête DNS ne prouve pas qu'une connexion a ensuite été établie. |

Une correspondance IP n'est pas une preuve d'accès à un site précis : plusieurs services peuvent partager une même adresse IP. Les caches DNS, les mécanismes DNS propres aux applications et l'indisponibilité des événements peuvent également limiter les observations. Certains processus terminés ou protégés peuvent ne pas exposer tous leurs détails.

Le mode **FULL capture** reste une collecte de métadonnées via les sources Windows, pas une capture de paquets exhaustive. L'outil ne déchiffre pas le trafic et ne collecte pas le contenu des échanges.

## Journaux CSV

Les fichiers sont créés dans le sous-dossier [`Logs`](Logs), à côté du script ou de l'exécutable :

```text
AppConnectionFinder_yyyyMMdd_HHmmss.csv
AppConnectionFinder_FULLCAPTURE_yyyyMMdd_HHmmss.csv
```

Colonnes exportées :

```text
Timestamp, Method, Target, MatchedValue, Detail, ProcessName, ProcessId, ProcessPath, Company
```

Le CSV enregistre les nouvelles détections, regroupées par méthode, cible, PID et valeur correspondante. Les observations répétées mettent à jour **Last seen** et **Count** dans le tableau, sans ajouter une ligne CSV à chaque passage. Le compteur représente des observations, pas un nombre de paquets ni de connexions uniques.

Si la création du fichier échoue, un avertissement apparaît dans **Activity log** ; la surveillance peut continuer sans journal sur disque.

## Modifications temporaires et confidentialité

Selon le mode utilisé, l'application peut activer temporairement le canal DNS Client Operational et l'audit de succès **Filtering Platform Connection**. Elle mémorise leur état initial et prévoit de le restaurer à l'arrêt de la surveillance, à la fermeture de la fenêtre et lors du nettoyage après une erreur.

Une terminaison forcée du processus ou un arrêt brutal de Windows peut empêcher ce nettoyage. Vérifiez alors les paramètres avec votre administrateur. L'activation de l'audit peut augmenter le volume du journal Security.

Les journaux peuvent contenir des noms de domaine, des adresses IP et des chemins d'exécutables sensibles. Utilisez l'outil uniquement sur des machines pour lesquelles vous disposez d'une autorisation et vérifiez les CSV avant de les partager ou de les ajouter à un dépôt.

---

## English

Identify which Windows applications communicate with a hostname or IP address through a graphical interface. AppConnectionFinder combines active TCP connections, Windows connection events and DNS queries to help troubleshoot network activity.

The application interface is in English.

### Features

- Targeted monitoring of one or more URLs, hostnames or IPv4/IPv6 addresses.
- Hostname resolution with periodic IP address refresh.
- Process identification: name, PID, executable path and company, when available.
- Correlation of TCP connections, UDP flows and DNS queries.
- Target-free **FULL capture** mode to observe activity across all applications.
- One-off TCP checks with **Check now (snapshot)**.
- Configurable polling interval from 1 to 30 seconds; default: 2 seconds.
- First/last observation times, observation counts and connection details.
- Automatic CSV logging during monitoring.

### Requirements

- Windows with an interactive desktop: the interface uses Windows Forms.
- For the script: Windows PowerShell 5.1 or PowerShell 7+ on Windows.
- Administrator rights for full functionality, including connection auditing, DNS correlation and some process details.
- Write access to the application directory to create log files.

The script uses built-in Windows components and does not require third-party PowerShell modules.

### Getting started

#### Executable

Run [`AppConnectionFinder.exe`](AppConnectionFinder.exe) from a writable directory and accept the UAC elevation prompt for full functionality.

#### PowerShell script

From the project directory:

```powershell
# Windows PowerShell 5.1
powershell.exe -NoProfile -STA -File .\AppConnectionFinder.ps1

# PowerShell 7+
pwsh.exe -NoProfile -STA -File .\AppConnectionFinder.ps1
```

If the execution policy blocks the script and your organization's rules permit it, use a process-scoped exception:

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass -STA -File .\AppConnectionFinder.ps1
```

The application attempts to relaunch with elevation when needed. If elevation is declined or fails, it remains available with reduced functionality; check **Activity log** for warnings.

### Usage

#### Monitor specific targets

1. Enter a URL, hostname or IP address in **URL / hostname or IP address**.
2. Click **Add target**. Multiple targets can be entered separated by commas.
3. Check the targets to monitor and set **Poll interval (seconds)**.
4. Click **Start logging**, then reproduce the network activity you want to investigate.
5. Review the processes and detection methods in the results grid.
6. Click **Stop logging**, then **Open log folder** to locate the CSV.

Example inputs:

```text
https://www.microsoft.com/en-us
example.com
192.0.2.10
2001:db8::10
example.com, 192.0.2.10
```

**Important:** URLs are reduced to their hostname or IP address. The supplied scheme, path and port are not filters: this is not a tool for tracking a specific HTTP URL.

#### Observe all applications

Click **Start FULL capture (all connections, no filter)** and confirm. No target is required. This mode collects visible TCP connections, including listening sockets, available UDP flow events and DNS queries without target filtering.

On a busy machine, this can generate substantial data. Keep the session limited to the time needed for troubleshooting.

#### Other controls

- **Check now (snapshot)** checks only active TCP connections matching the checked targets. Use continuous monitoring for UDP and DNS events.
- **Clear results** clears the grid without deleting or emptying the existing CSV file.
- **Remove selected / Clear all** removes selected targets or all targets.

### Detection methods and limitations

| Method | Source | Interpretation |
|---|---|---|
| TCP | `Get-NetTCPConnection`, with a `netstat` fallback | Matches remote addresses against target IPs and identifies the owning process. Very short-lived connections may occur between polls. |
| UDP | Security log, Windows Filtering Platform event 5156 | Uses permitted-connection auditing to identify the process and flow endpoints. Requires access to the log and audit policy. |
| DNS | `Microsoft-Windows-DNS-Client/Operational`, events 3006/3008 | Associates a DNS query with the process reported by Windows. A DNS query does not prove that a connection was subsequently established. |

An IP match does not prove access to a specific website: multiple services can share an IP address. DNS caches, application-specific DNS mechanisms and unavailable events can also limit visibility. Details may be incomplete for exited or protected processes.

**FULL capture** collects metadata from Windows sources; it is not an exhaustive packet capture. The tool does not decrypt traffic or collect communication payloads.

### CSV logs

Files are created in the [`Logs`](Logs) subdirectory next to the script or executable:

```text
AppConnectionFinder_yyyyMMdd_HHmmss.csv
AppConnectionFinder_FULLCAPTURE_yyyyMMdd_HHmmss.csv
```

Exported columns:

```text
Timestamp, Method, Target, MatchedValue, Detail, ProcessName, ProcessId, ProcessPath, Company
```

The CSV records new detections grouped by method, target, PID and matched value. Repeated observations update **Last seen** and **Count** in the grid without appending a CSV row on every poll. The count represents observations, not packets or unique connections.

If file creation fails, **Activity log** displays a warning; monitoring may continue without a log file on disk.

### Temporary changes and privacy

Depending on the monitoring mode, the application may temporarily enable the DNS Client Operational channel and **Filtering Platform Connection** success auditing. It saves their initial state and includes restoration logic when monitoring stops, the window closes or cleanup runs after an error.

Forcefully terminating the process or abruptly shutting down Windows may prevent cleanup. In that case, review the settings with your administrator. Enabling auditing may increase Security log volume.

Logs may contain sensitive domain names, IP addresses and executable paths. Use the tool only on machines you are authorized to inspect, and review CSV files before sharing them or adding them to a repository.
