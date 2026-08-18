---
title: "Deploying Wazuh on Xubuntu 24.04 and Enrolling a Windows 10 OT Station"
date: 2026-08-18 00:32:00 +0100
project: "ICS-Security"
categories: [Cybersecurity, ICS-OT]
tags: [Wazuh, Sysmon, Xubuntu, Windows10, deployment, OT-monitoring, agent-enrollment]
description: "Step-by-step deployment of the full Wazuh stack on Xubuntu 24.04, agent enrollment on a Windows 10 OT engineering station, and Sysmon integration for advanced event collection."
---

Architecture decisions are easy to document. Getting the software to actually run 
is where things get interesting. This article covers the full deployment process: 
Wazuh on Xubuntu 24.04, agent enrollment on the Windows 10 OT station, and Sysmon 
configuration so the detection chain works end to end.

A few things happened during this installation that are worth documenting honestly, 
because they will happen to anyone who follows the same path.

## Installing Wazuh on Xubuntu 24.04

Wazuh provides an all-in-one installation script that handles the Manager, Indexer, 
and Dashboard in a single run. Before launching it, the official Wazuh repository 
needs to be added to the package manager along with its GPG key.

Once the repository is in place, the installation script takes over. It generates 
SSL certificates, initializes the OpenSearch cluster, configures the Dashboard, 
and starts all three services. The process takes between 15 and 20 minutes depending 
on the machine. At the end, the terminal prints the generated admin credentials. 
Those credentials are only shown once, so they need to be saved immediately.

The first thing worth noting: Xubuntu 24.04 is not on Wazuh's official list of 
supported distributions. The script will refuse to run without the `-i` flag, which 
bypasses the OS compatibility check. In practice, Xubuntu 24.04 is Debian-based and 
everything works cleanly. The flag is just a formality here.

The second thing worth noting: during the installation, the Ubuntu automatic update 
service (`unattended-upgrades`) woke up and locked the APT frontend. The Wazuh 
script retried every 30 seconds up to ten times before failing and rolling back the 
entire installation, including the already-installed Indexer. The fix was to disable 
`unattended-upgrades` before running the script again. That is not documented 
anywhere obvious, but it will catch you if you are not expecting it.

Once the installation completes cleanly, three systemd services should be active:

![Wazuh Manager Active](/assets/img/posts/wazuh_project/systemctl_waz_manager.png)
*Figure 1 - Wazuh Manager running on the supervision server.*

![Wazuh Indexer Active](/assets/img/posts/wazuh_project/systemctl_waz_indexer.png)
*Figure 2 - Wazuh Indexer running on the supervision server.*

![Wazuh Dashboard Active](/assets/img/posts/wazuh_project/systemctl_waz_dashboard.png)
*Figure 3 - Wazuh Dashboard running and accessible via HTTPS on port 443.*

The Dashboard is accessible at `https://192.168.100.10`. The browser will flag the 
self-signed certificate. That is expected. Proceed past the warning.

![Wazuh Dashboard Login](/assets/img/posts/wazuh_project/waz_dashb_before_login.png)
*Figure 4 - The Wazuh Dashboard authentication page.*

![Wazuh Dashboard Home](/assets/img/posts/wazuh_project/waz_dashb_After_login.png)
*Figure 5 - The Dashboard home view after authentication. No agents connected yet.*

## Enrolling the Windows 10 OT Station

Agent enrollment starts from the Dashboard. Under the Agents section, the "Deploy 
new agent" wizard generates a customized installation command that already contains 
the Manager's IP address and the agent name. Select Windows as the target platform, 
set the server address to `192.168.100.10`, name the agent `Station-OT`, and the 
wizard produces the installation command.

![Deploy New Agent Interface](/assets/img/posts/wazuh_project/screen7_deploy_wind_agent.png)
*Figure 6 - The Deploy New Agent wizard in the Wazuh Dashboard with Windows selected 
and the Manager IP pre-populated.*

The OT station has no internet access, so the MSI installer cannot be downloaded 
directly from the Wazuh repository. The solution was to download it on the Xubuntu 
server first, then serve it temporarily over HTTP using a one-line Python server. 
The Windows machine fetched it from `http://192.168.100.10:8080` and the 
installation ran from PowerShell with administrator privileges.

Once the service starts, the agent registers itself with the Manager and appears 
in the Dashboard within seconds.

![Station-OT Active in Dashboard](/assets/img/posts/wazuh_project/screen_08_agent_active_dashboard.png)
*Figure 7 - Station-OT connected and active in the Wazuh Dashboard. Agent ID 001.*

An active agent in the Dashboard confirms four things at once: the agent service 
is running on Windows, TCP 1514 is open between the two machines, the agent 
authenticated successfully against the Manager, and events are flowing.

## Integrating Sysmon for Advanced Event Collection

A connected agent alone is not enough. Windows native event logs give Wazuh 
process creation events and some security events, but they do not expose network 
connections per process, file creation with process attribution, or registry 
modifications in a format that is easy to work with. Sysmon fills all of those gaps.

The installation uses the SwiftOnSecurity configuration file, which is the 
community standard for Sysmon deployments. It was designed to maximize coverage of 
security-relevant events while filtering out the constant background noise that 
makes raw Sysmon logs unusable in practice. Without that filtering, the volume 
of events would overwhelm the detection rules and make the Dashboard unreadable.

Sysmon is transferred to the Windows machine the same way the Wazuh MSI was: 
downloaded on the Xubuntu server, served over HTTP, fetched on Windows. The 
installation runs from PowerShell with the configuration file passed as an argument.

![Sysmon Running on OT Station](/assets/img/posts/wazuh_project/screen_09_sysmon_running.png)
*Figure 8 - Sysmon64 service active on the Windows 10 OT engineering station.*

Installing Sysmon is only half the work. The Wazuh agent needs to know that this 
new event channel exists. By default, it collects standard Windows event logs. 
To add the Sysmon channel, the agent configuration file `ossec.conf` needs a new 
`localfile` block pointing to `Microsoft-Windows-Sysmon/Operational`. After 
restarting the agent service, Sysmon events start flowing to the Manager.

## Verifying the Full Collection Chain

The verification step is simple but essential. In the Dashboard, navigate to the 
Station-OT agent view and open the Security Events tab. Sysmon events are 
identifiable by their channel name and their Event IDs. If Event ID 1 (ProcessCreate) 
and Event ID 3 (NetworkConnect) events are appearing with process-level detail, 
the chain is working correctly.

![Sysmon Events in Dashboard](/assets/img/posts/wazuh_project/screen_10_sysmon_logs_detail.png)
*Figure 9 - Sysmon events reaching the Wazuh Dashboard. The enriched fields 
(process image, command line, parent process) confirm that Sysmon is feeding 
the agent correctly.*

![Real-Time Events from Station-OT](/assets/img/posts/wazuh_project/screen_11_events_station_ot.png)
*Figure 10 - Real-time event stream from the OT engineering station in the Wazuh 
Dashboard. The collection chain is fully operational.*

At this point the infrastructure is complete. The supervision server is running, 
the OT station is enrolled, Sysmon is collecting, and events are reaching the 
Dashboard in real time. What is missing is the detection logic on top of this 
collection layer.

The next article covers the attack scenario itself: what the OT station contains, 
why those assets are worth protecting, and how the attack vector was designed to 
reproduce the behavioral fingerprint of a real ransomware without using one.

