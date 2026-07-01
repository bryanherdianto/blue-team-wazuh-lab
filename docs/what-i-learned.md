# What I Learned — Wazuh Blue Team Lab

## Table of Contents

1. [Docker Deployment](#1-docker-deployment)
   - [Persistent sysctl Setting](#persistent-sysctl-setting)
   - [VPS Resources & Dashboard Access](#vps-resources--dashboard-access)
2. [Wazuh Agent Deployment](#2-wazuh-agent-deployment)
   - [Windows Agent](#windows-agent)
   - [Ubuntu Agent](#ubuntu-agent)
3. [Telemetry: Event Monitor vs Sysmon vs Wazuh](#3-telemetry-event-monitor-vs-sysmon-vs-wazuh)
4. [Custom Detection Rules](#4-custom-detection-rules)
   - [Rule Syntax](#rule-syntax)
   - [DeerStealer Malware](#deerstealer-malware)
5. [Detecting cipher.exe](#5-detecting-cipherexe)
6. [File Integrity Monitoring (FIM)](#6-file-integrity-monitoring-fim)
7. [VirusTotal Integration](#7-virustotal-integration)
8. [Active Response](#8-active-response)
   - [SSH Setup for Ubuntu VM](#ssh-setup-for-ubuntu-vm)
   - [VirtualBox Network Modes](#virtualbox-network-modes)
   - [remove-threat Script](#remove-threat-script)
   - [File Ownership (chown) & Permissions (chmod)](#file-ownership-chown--permissions-chmod)
9. [MariaDB Auditing](#9-mariadb-auditing)
   - [Installation (Ubuntu 22.04)](#installation-ubuntu-2204)
   - [Audit Configuration](#audit-configuration)
   - [Decoders & Rules](#decoders--rules)
10. [MITRE ATT&CK Framework](#10-mitre-attck-framework)
11. [SBOM & Dependency-Track](#11-sbom--dependency-track)
12. [Wazuh OVA](#12-wazuh-ova)
13. [TTP Framework](#13-ttp-framework)
14. [Zeek Network Monitoring](#14-zeek-network-monitoring)
15. [FIM Configuration Note](#15-fim-configuration-note)

---

## 1. Docker Deployment

So, I used docker deployment instead of manually deploying wazuh manager, wazuh indexer, and wazuh dashboard. This is because independent download may need independent dependencies and configuration that can be cumbersome. Docker deployment simplifies the process by encapsulating all necessary components and their dependencies into containers, making it easier to manage and deploy. Also, there are more steps to follow in the manual deployment, which can be time-consuming and error-prone, like initial config, node install, cluster init, and testing the cluster. Docker deployment, on the other hand, allows for a more streamlined setup process, reducing the potential for mistakes and saving time.

### Persistent sysctl Setting

Also, the documents asked me to `sysctl -w vm.max_map_count=262144`... but this has a flaw in the sense that it is not persistent across reboots. To make it persistent, I had to add the line `vm.max_map_count=262144` to `/etc/sysctl.conf` and then run `sysctl -p` to apply the changes. This ensures that the setting remains in effect even after a system restart. This is the full command:

```
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

Also, it's supposed to be `docker-compose`, not `docker compose`.

Alas, running `docker-compose up -d` is mandatory I guess, I mean like it's the default, you can omit the `-d` if you want to see the logs in the terminal, but if you want to run it in the background, `-d` is necessary.

`generate-indexer-certs` is used to generate TLS certificates for secure communication between the Wazuh components. This is important for ensuring that data transmitted between the manager, indexer, and dashboard is encrypted and secure. The command typically creates the necessary certificate files and places them in the appropriate directories for use by the Wazuh services.

### VPS Resources & Dashboard Access

I can use `free -h` to see current total ram and available ram. Even though, the document says cpu needed is 4, but my current vps has spec 2 vCPU and 8GB RAM... and it still works, so for my small scale lab, it is sufficient. But for production, I guess 4 vCPU and 8GB RAM is the minimum requirement.

`docker ps` and `docker-compose ps` are different because `docker ps` shows all running containers on the system, while `docker-compose ps` shows the status of containers defined in the docker-compose.yml file for a specific project. This distinction is important for managing and monitoring containers in a multi-container environment.

You can see the public ip of your vps by using the command `curl ifconfig.me`. This command queries an external service to return the public IP address of your VPS, which can be useful for accessing services running on the VPS from outside networks.

Because docker set this `0.0.0.0:443->5601/tcp`, then it means that by accessing the host's IP on port 443 or just HTTPS, you can reach the Wazuh dashboard. The mapping indicates that the container's port 5601 is exposed to the host's port 443, allowing secure access to the dashboard via HTTPS. The user and password for the wazuh dashboard is username is `admin` and password is `SecretPassword`.

---

## 2. Wazuh Agent Deployment

For this lab, I'm gonna install wazuh agent on my 2 vms. One vm is windows, while the other is ubuntu. The Wazuh agent is responsible for collecting and sending security data from the endpoints to the Wazuh manager for analysis. Installing the agent on both Windows and Ubuntu VMs allows for monitoring and securing different operating systems within the lab environment. Later, I can install vulnerabilities or malware in the VMs to test the detection capabilities of the Wazuh setup. This will help in evaluating how well the Wazuh system can identify and respond to security threats across different platforms.

### Windows Agent

To deploy agent in the windows:

1. Set package to download to be Windows the MSI 32/64 bits.
2. Set the server address to be the public ip of the vps where the wazuh manager is running.
3. Set agent name into `windows10` (since it's OS is Windows 10)
4. After that, just copy paste the commands that Wazuh has provided on Powershell with Administrator privileges
5. Do `NET START Wazuh`
6. If they already show up, then good. If not, then do hard refresh on wazuh dashboard by `Ctrl + Shift + R`.

### Ubuntu Agent

To deploy the agent in the ubuntu:

1. Set package to download to be Linux the DEB and AMD64 architecture. For reference, DEB includes ubuntu, debian, and other debian-based distributions. Meanwhile, RPM includes centos, fedora, and other redhat-based distributions.
2. Set the server address to be the public ip of the vps where the wazuh manager is running.
3. Set agent name into `ubuntu22-server` (since it's OS is Ubuntu 22.04 server)
4. After that, just copy paste the commands that Wazuh has provided on terminal with root privileges (used sudo in front of all the commands)
5. Again with sudo, do `systemctl daemon-reload`, `systemctl enable wazuh-agent`, and `systemctl start wazuh-agent`.
6. If they already show up, then good. If not, then do hard refresh on wazuh dashboard by `Ctrl + Shift + R`.

---

## 3. Telemetry: Event Monitor vs Sysmon vs Wazuh

Event monitor, sysmon, and wazuh are different tools for monitoring and analyzing system events and security incidents. Event monitor can only show the events that are logged by the system, while sysmon is a more advanced tool that can provide detailed information about system activity, including process creation, network connections, and file modifications. Wazuh, on the other hand, is a comprehensive security monitoring platform that can collect and analyze data from multiple sources, including event logs, sysmon data, and other security tools. It provides a centralized view of security events and can generate alerts based on predefined rules and policies.

Telemetry means the process of collecting and transmitting data from remote sources to a central system for monitoring and analysis. In the context of sysmon, it gives more telemetry data than event monitor, which can be useful for detecting and investigating security incidents. This is also the same for auditd in Linux, which provides detailed auditing capabilities for monitoring system activity and security events. By collecting telemetry data from various sources, security teams can gain insights into potential threats and take proactive measures to protect their systems and networks.

Sysmon works with XML file to config. There is 2 common types of config (but still doesn't change the fact you can customize it), that is by using SwiftOnSecurity and Olaf Hartong's sysmon-modular. Sysmon config file works by whitelisting or blacklisting certain events, processes, or behaviors based on the defined rules. The SwiftOnSecurity config is a widely used configuration that provides a good balance between security and usability, while Wazuh's own config is tailored to work seamlessly with the Wazuh platform, allowing for better integration and detection capabilities. Users can customize these configs to suit their specific security needs and environment. Auditd is similar to sysmon in that it provides detailed auditing capabilities for monitoring system activity and security events on Linux systems. It can be configured to log specific events, processes, or behaviors, allowing for granular control over what is monitored and reported. Both sysmon and auditd are valuable tools for enhancing security monitoring and incident response efforts.

`ossec.conf` is the main configuration file for Wazuh, which defines various settings and parameters for the Wazuh manager and agents. We can find this file in `Program Files (x86)/ossec-agent`. It includes configurations for log collection, alerting, rules, decoders, and other components of the Wazuh platform. The `ossec.conf` file allows administrators to customize the behavior of Wazuh to meet their specific security monitoring requirements. It is important to carefully configure this file to ensure that the Wazuh system operates effectively and efficiently in detecting and responding to security threats. Like for sysmon, I need to set the config file so that wazuh can see the events that sysmon has logged. After using, `sysmon64` is the exe used to install sysmon on windows with 64 bit architecture. Then after that, we can change the `ossec.conf`. BTW this is done on Wazuh agent, so on my VM windows 10.

I can use `Ctrl + Shift + enter` to enter the app in administrator mode, which is useful for performing tasks that require elevated privileges, such as installing software, modifying system settings, or running commands that affect the entire system. This shortcut provides a quick and convenient way to launch applications with administrative rights without having to navigate through menus or use the right-click context menu.

---

## 4. Custom Detection Rules

Right next, I'm trying out detecting deerstealer malware with wazuh. I can do it by creating the new file, but I can also access it via GUI from the Wazuh dashboard > going to the server management > create new rule. Here's how the rule syntax works:

### Rule Syntax

- `rule` tag has a unique id number (if you make it within the range 100000 to 120000) and the level shows how serious it is. Like, 0 is for ignore, 3-6 to low/medium, and 7-11 is high, while 12 above is critical.
- `if_sid` is like only fire this rule if the parent rule with id `if_sid` has fired. This is useful for creating hierarchical rules where certain conditions must be met before a rule is triggered. So, it's like a chained condition.
- `field` tag means matching the field in the log. For example, if I want to match the field "srcip" in the log, then I can use field tag with `name="srcip"` and then set the value to be matched. Usually, it's used with pcre2 or regex for the matching.
- `description` shows the human text shown in alert if the regex matches.
- `mitre` maps the alert to which mitre attack technique it belongs to. This will show the technique ID and name.
- `group` is used to categorize the rule into a specific group or category, which can help in organizing and managing rules within the Wazuh platform. We use comma to separate multiple groups if needed.

### DeerStealer Malware

AnyRun is needed so I can download the malware sample, which is deerstealer malware. AnyRun is an interactive online malware analysis sandbox that allows users to safely analyze and observe the behavior of malicious files in a controlled environment. By using AnyRun, I can obtain the necessary malware samples for testing and detection purposes without risking the security of my own systems. So, Deerstealer malware is a malware that obtains password from web browsers and take crypto money, then sent it to the attacker. It's not a worm, so it doesn't spread to the network and only effects the host where the exe is run. But, it might use the vulnerability in virtualbox to execute in host laptop, and if there are any shared resources. Btw, VM isn't always safe or isolated if the malware can spread to other devices in a network.

---

## 5. Detecting cipher.exe

So, instead of running deerstealer, I tried to make alert for running `cipher.exe`. `Cipher.exe` is a legitimate Windows utility used for managing encryption on NTFS volumes. However, it can also be exploited by attackers to perform malicious activities, such as encrypting files for ransomware attacks. By creating an alert for the execution of `cipher.exe`, I can monitor and detect any suspicious usage of this utility, which may indicate potential security threats or unauthorized actions on the system. In the process of creating the rule, I realize that process creation or event ID 1 is not being included in the whitelist of sysmon config, so I need to add it in manually. Previously, only event ID 10 is being logged for `cipher.exe` in which powershell is calling `cipher.exe`. After fixing this, now I can see the alert for `cipher.exe` from event ID 1, which is the process creation event. Why monitoring for event ID 1 is better than event ID 10 is because event ID 1 is the first event that occurs when a process is created, while event ID 10 is generated later in the process lifecycle. By monitoring for event ID 1, I can detect the execution of `cipher.exe` as soon as it starts, allowing for quicker response and mitigation of potential threats. Meanwhile, if I only monitor event ID 10, I might miss the initial execution of `cipher.exe`, which could delay detection and response to malicious activity. Also, there might be multiple event ID 10 for the same process, which can lead to confusion and make it harder to identify the actual threat.

I use `.\sysmon64.exe -c sysmonconfig.xml` to update the sysmon config. The `-c` option specifies the configuration file to be used, allowing me to apply the new settings defined in `sysmonconfig.xml`. This command updates the existing sysmon configuration with the rules and parameters specified in the provided XML file.

---

## 6. File Integrity Monitoring (FIM)

Next, I learned about FIM (File Integrity Monitoring). **I DID THIS ON MY UBUNTU VM.** Basically, it checks if a file has been modified or not by tracking the checksums of the files. If the checksum changes, it means the file has been modified. This is useful for detecting unauthorized changes to critical system files, configuration files, or other important data. FIM can help identify potential security breaches, malware infections, or accidental modifications that could compromise the integrity of the system. By monitoring file integrity, organizations can ensure that their systems remain secure and compliant with security policies and regulations.

To do this, I need to modify `agent.conf` file in wazuh. I used this on os Linux and configured yes on `check_all` and `realtime`. Realtime means the alert will come immediately after the file is modified, while `check_all` means it will check every attribute of the files, like `check_sum`, `check_size`, `check_owner`, `check_perm`, etc. This ensures that any changes to the monitored files are detected and reported promptly, allowing for timely investigation and response to potential security incidents. For the address, it needs to be absolute path, to get the `~` mean I need to `echo $HOME`.

---

## 7. VirusTotal Integration

Next, for virustotal integration, I need to use virustotal api key and change it on `ossec.conf` on the wazuh server (BTW docker has the conf file on the docker conf folder, not on var folder of the host). The key will be added to the `ossec.conf` file in the manager. This means if there are file changes in the FIM, then the wazuh manager will send the file to virustotal for scanning. If virustotal returns a positive result, then the wazuh manager will generate an alert. This integration allows for automated malware detection and enhances the overall security monitoring capabilities of the Wazuh platform. Because I used docker, then the `wazuh_manager.conf` file needs to be changed, then after that `docker-compose down` and up again to apply the changes. No need to `systemctl restart wazuh-manager` because the `docker-compose down` and up will restart the wazuh manager container.

---

## 8. Active Response

To enable active response, I need to install `jq` library in the wazuh agent. `jq` library is used to process JSON data from wazuh then implement automated response actions. Also, I just realized that it's better to ssh to the ubuntu server vm rather than installing guest additions (which I don't know the manual). Here's the step to enable ssh in the vm:

### SSH Setup for Ubuntu VM

```
sudo apt install -y openssh-server
sudo systemctl enable --now ssh
sudo ss -tlnp | grep :22
ip -4 addr show | grep inet
```

### VirtualBox Network Modes

Also the vm must use bridged adapter, not NAT, so that the vm can be accessed from the host machine. So the different modes of network in virtualbox are:

- **NAT (Network Address Translation):** The VM shares the host's IP address and can access external networks, but it is not directly accessible from the host or other devices on the network.
- **NAT network:** Similar to NAT, but allows multiple VMs to communicate with each other within a private network while still sharing the host's IP address for external access.
- **Bridged adapter:** The VM is connected directly to the physical network, allowing it to have its own IP address and be accessible from other devices on the same network, including the host machine.
- **Internal network:** The VM can communicate only with other VMs on the same internal network, but it cannot access external networks or the host machine.
- **Host-only adapter:** The VM can communicate only with the host machine and other VMs configured with the same host-only network, but it cannot access external networks.
- **Generic driver:** This option allows for the use of third-party network drivers, which may provide additional features or compatibility with specific network configurations.

### remove-threat Script

Also edit the docker wazuh manager conf file again:

```xml
<command>
  <name>remove-threat</name>
  <executable>remove-threat.sh</executable>
  <timeout_allowed>no</timeout_allowed>
</command>

<active-response>
  <disabled>no</disabled>
  <command>remove-threat</command>
  <location>local</location>
  <rules_id>87105</rules_id>
</active-response>
```

### File Ownership (chown) & Permissions (chmod)

`chown` is used to change the ownership of the file. In this case, I need to change the ownership of `remove-threat.sh` to root user and root group. This is important for security reasons, as it ensures that only authorized users have access to the script and can execute it. By setting the correct ownership, I can prevent unauthorized modifications or execution of the script, which could potentially compromise the security of the system.

`chmod` to 750 for the `remove-threat.sh` file is used to set the file permissions, allowing the owner (root) to read, write, and execute the script, while the group (root) can read and execute it, and others have no permissions. This permission setting ensures that only authorized users can access and execute the script, enhancing the security of the system. Aside from 750, there is also 755, which allows the owner to read, write, and execute the script, while the group and others can read and execute it.

---

## 9. MariaDB Auditing

Next, still on ubuntu vm, I want to download MariaDB. Because I'm using ubuntu server, instead of fedora as stated in the document, I must tweak a few things.

### Installation (Ubuntu 22.04)

```
sudo apt install mariadb-server mariadb-client galera-4
sudo mariadb-secure-installation
sudo systemctl status mariadb
sudo mariadb -u root
```

### Audit Configuration

Then after that, we can add the wazuh config to the mariadb (this is the part where it's different from the docs):

```
sudo nano /etc/mysql/mariadb.conf.d/50-audit.cnf
<copy paste the content of 50-audit.cnf from my directory>
sudo systemctl restart mariadb
sudo tail -f /var/log/syslog | grep mariadb-server_auditing (check if its working)
```

### Decoders & Rules

Wazuh has thousands of default rules. If there were no decoders, every time a single log arrived, the manager would have to run complex regex operations against every single rule to see if it matched. Your CPU would max out at 100% almost immediately, and log processing would grind to a halt.

- **Without Decoders:** You would have to write complex PCRE2 regex in your rule to account for the Windows format, the Linux format, the Cisco format, etc. You would need dozens of duplicate rules.
- **With Decoders:** The Windows decoder, the Linux decoder, and the Cisco decoder all translate their unique logs into the exact same variable: `srcip`. Now, you only have to write one single rule that says: "If `srcip` fails 5 times, trigger an alert."

Ok after inserting the mariadb decoders and rules to the wazuh manager. BTW I used the GUI instead of manually inserting into `/var/ossec/etc/rules` and `/var/ossec/etc/decoders`.

Now because that's already made, I can monitor multiple activities from MariaDB. Like authentication operations, data definition operations, data manipulation operations, data destruction operations, data access and query operations, account and privilege operations.

---

## 10. MITRE ATT&CK Framework

### 1. The Headers = Tactics (The Attacker's Goal)

The big bold headers at the top (like Initial Access, Execution, Persistence) are called **Tactics**.

Think of these as the attacker's high-level objectives. It answers the question: "What is the hacker trying to achieve at this exact moment?"

For example, under Initial Access, the hacker's goal is simply to get a foot in the door of your network. Under Persistence, their goal is to make sure they don't lose access if a server reboots.

### 2. The Cells Below = Techniques (The Attacker's Method)

The individual boxes underneath each header (like Drive-by Compromise, BITS Jobs, Command and Scripting Interpreter) are called **Techniques**.

Think of these as the specific actions or methods the hacker uses to accomplish that top header goal. It answers the question: "How exactly is the hacker pulling this off?"

For example, if the attacker's goal is Execution (running malicious code on your server), one technique they might use to do that is a BITS Jobs or a Command and Scripting Interpreter (like running a malicious PowerShell or Bash script).

### Where does the "Defense" come in?

The Navigator starts completely grey because it only lists the hacker's playbook. The defense part is where you have to do it yourself (either manually or automatically). So you color code it, like red for the techniques that you have detected, yellow for the techniques that you have partially detected, and green for the techniques that you have fully detected. This allows you to visualize your defensive coverage against the attacker's methods and identify areas where you may need to improve your security posture. By mapping your detections to the MITRE ATT&CK framework, you can better understand the tactics and techniques used by attackers and enhance your overall threat detection and response capabilities.

One thing to note, is for example BITS Jobs is technique that comes under the tactic Execution, Persistence, and stealth. This means that if I color code BITS Jobs, then it will be colored in all 3 tactics. This is because the same technique can be used by attackers to achieve multiple objectives, and by mapping it to multiple tactics, I can gain a more comprehensive understanding of the attacker's methods and potential impact on my systems.

---

## 11. SBOM & Dependency-Track

SBOM can be generated for project web apps or mobile apps. SBOM stands for Software Bill of Materials, which is a comprehensive list of all the components, libraries, and dependencies used in a software application. It provides transparency into the software supply chain and helps organizations identify potential security vulnerabilities or licensing issues associated with the components used in their applications. By generating an SBOM, developers and security teams can better understand the composition of their software and take proactive measures to mitigate risks and ensure compliance with security standards and regulations.

```
npm install --save-dev @cyclonedx/cyclonedx-npm
```

`--save-dev` is used for development dependencies.

Now, command the tool to scan your project and spit out a JSON-formatted bill of materials:

```
npx @cyclonedx/cyclonedx-npm --output-format JSON --output-file bom.json
```

Or if using python, then:

```
pip install cyclonedx-bom
cyclonedx-py -o backend-bom.json
```

Example of bom is `game-centr-frontend-bom.json`. This file is from a frontend of my project game centr.

After that, the wazuh way is by uploading that SBOM to dependency track and using wazuh to generate the alerts. Dependency-Track is an open-source software composition analysis (SCA) platform that helps organizations identify and manage security vulnerabilities in their software dependencies. By uploading the SBOM to Dependency-Track, I can analyze the components used in my project and receive alerts for any known vulnerabilities or security issues associated with those components. Wazuh can then be configured to monitor the Dependency-Track platform and generate alerts based on the findings, allowing for proactive management of software security risks. This integration enhances the overall security posture of the application by providing visibility into potential threats and enabling timely remediation actions.

---

## 12. Wazuh OVA

There is also Wazuh OVA (open virtual appliance). So this wazuh is already pre-installed with wazuh dashboard, wazuh manager, and wazuh indexer. This is useful for quickly setting up a Wazuh environment without having to go through the manual installation and configuration process. The OVA can be imported into virtualization platforms like VMware or VirtualBox, allowing for easy deployment and testing of Wazuh in a controlled environment. It provides a convenient way to evaluate the capabilities of Wazuh and its components without the need for extensive setup or configuration.

---

## 13. TTP Framework

In cybersecurity, TTP stands for:

- **Tactics** → the goal or objective of an attack
- **Techniques** → how the attacker achieves that goal
- **Procedures** → the specific steps/tools they actually use

Think of it like this:

- Tactic = Why
- Technique = How
- Procedure = Exactly what they did

**Example:**

Imagine an attacker wants to steal company files.

- **Tactic:** Credential Access (get login credentials)
- **Technique:** Phishing email
- **Procedure:** Send fake Microsoft login page → collect password → log into VPN

---

## 14. Zeek Network Monitoring

Wazuh is an open source SIEM/XDR platform that monitors, detects, and responds to security threats across endpoints, networks, and cloud environments in real time. Zeek, on the other hand, is a Linux-based open source framework for network visibility that analyzes traffic and produces structured logs and artifacts describing observed activity. It's for Network monitoring / network telemetry, it mostly observes and records. By ingesting Zeek logs, Wazuh enriches them with security context and applies correlation, alerting, and automated response actions. Integrating Zeek with Wazuh delivers unified, actionable visibility into network events and strengthens an organization's ability to detect, investigate, and respond to security threats.

The docs for installing zeek is using ubuntu 24. Because I'm using ubuntu 22, I need to tweak a few things:

```
curl -fsSL https://download.opensuse.org/repositories/security:zeek/xUbuntu_22.04/Release.key | \
gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/security_zeek.gpg > /dev/null
echo 'deb http://download.opensuse.org/repositories/security:/zeek/xUbuntu_22.04/ /' | \
sudo tee /etc/apt/sources.list.d/security:zeek.list
sudo apt update
sudo apt install zeek -y
```

Another error I found is that `zeekctl` was not in my system PATH, so I had to use the full path:

```
/opt/zeek/bin/zeekctl
```

Also I did fix the network interface (MAIN FIX). I changed Zeek from (this is because `eth0` is not found when I set `ip a` command):

```
eth0
```

to:

```
enp0s3
```

BTW sometimes the password for the Wazuh OVA is actually `admin`, not `SecretPassword`.

---

## 15. FIM Configuration Note

FIM is configured through agent or centralized syscheck configuration (`agent.conf`), while the Endpoint Security FIM tab only displays the alerts generated by the syscheck engine after the agent has applied those rules and detected file changes.
