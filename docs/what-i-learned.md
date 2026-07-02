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

I used Docker deployment instead of manually deploying the Wazuh manager, Wazuh indexer, and Wazuh dashboard. This is because deploying each component independently would require handling separate dependencies and configurations, which can be cumbersome. Docker deployment simplifies the process by encapsulating all necessary components and their dependencies into containers, making it easier to manage and deploy. There are also more steps to follow in the manual deployment, which can be time-consuming and error-prone, such as initial config, node install, cluster init, and testing the cluster. Docker deployment, on the other hand, allows for a more streamlined setup process, reducing the potential for mistakes and saving time.

### Persistent sysctl Setting

The documentation asked me to run `sysctl -w vm.max_map_count=262144`, but this has a flaw: it is not persistent across reboots. To make it persistent, I had to add the line `vm.max_map_count=262144` to `/etc/sysctl.conf` and then run `sysctl -p` to apply the changes. This ensures that the setting remains in effect even after a system restart. The full command is:

```
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

Also, it's supposed to be `docker-compose`, not `docker compose`.

Running `docker-compose up -d` is the standard approach. The `-d` flag is the default for background mode; you can omit it if you want to see the logs in the terminal, but if you want to run it in the background, `-d` is necessary.

`generate-indexer-certs` is used to generate TLS certificates for secure communication between the Wazuh components. This is important for ensuring that data transmitted between the manager, indexer, and dashboard is encrypted and secure. The command typically creates the necessary certificate files and places them in the appropriate directories for use by the Wazuh services.

### VPS Resources & Dashboard Access

I can use `free -h` to see the current total RAM and available RAM. Although the documentation states that 4 vCPUs are required, my current VPS has 2 vCPUs and 8GB of RAM, and it still works. For my small-scale lab, this is sufficient. For production, however, 4 vCPUs and 8GB of RAM is the minimum requirement.

`docker ps` and `docker-compose ps` are different because `docker ps` shows all running containers on the system, while `docker-compose ps` shows the status of containers defined in the docker-compose.yml file for a specific project. This distinction is important for managing and monitoring containers in a multi-container environment.

You can see the public IP of your VPS by using the command `curl ifconfig.me`. This command queries an external service to return the public IP address of your VPS, which can be useful for accessing services running on the VPS from outside networks.

Because Docker set up the mapping `0.0.0.0:443->5601/tcp`, accessing the host's IP on port 443 (or just HTTPS) allows you to reach the Wazuh dashboard. The mapping indicates that the container's port 5601 is exposed on the host's port 443, allowing secure access to the dashboard via HTTPS. The username and password for the Wazuh dashboard are `admin` and `SecretPassword`, respectively.

---

## 2. Wazuh Agent Deployment

For this lab, I installed the Wazuh agent on my two VMs. One VM is Windows, while the other is Ubuntu. The Wazuh agent is responsible for collecting and sending security data from the endpoints to the Wazuh manager for analysis. Installing the agent on both Windows and Ubuntu VMs allows for monitoring and securing different operating systems within the lab environment. Later, I can install vulnerabilities or malware in the VMs to test the detection capabilities of the Wazuh setup. This will help in evaluating how well the Wazuh system can identify and respond to security threats across different platforms.

### Windows Agent

To deploy the agent on Windows:

1. Set the package to download as the Windows MSI 32/64-bit version.
2. Set the server address to the public IP of the VPS where the Wazuh manager is running.
3. Set the agent name to `windows10` (since the OS is Windows 10).
4. After that, copy and paste the commands that Wazuh has provided into PowerShell with Administrator privileges.
5. Run `NET START Wazuh`.
6. If the agent already shows up, then good. If not, do a hard refresh on the Wazuh dashboard with `Ctrl + Shift + R`.

### Ubuntu Agent

To deploy the agent on Ubuntu:

1. Set the package to download as the Linux DEB package with AMD64 architecture. For reference, DEB covers Ubuntu, Debian, and other Debian-based distributions. Meanwhile, RPM covers CentOS, Fedora, and other Red Hat-based distributions.
2. Set the server address to the public IP of the VPS where the Wazuh manager is running.
3. Set the agent name to `ubuntu22-server` (since the OS is Ubuntu 22.04 server).
4. After that, copy and paste the commands that Wazuh has provided into the terminal with root privileges (I used `sudo` in front of all the commands).
5. Again with `sudo`, run `systemctl daemon-reload`, `systemctl enable wazuh-agent`, and `systemctl start wazuh-agent`.
6. If the agent already shows up, then good. If not, do a hard refresh on the Wazuh dashboard with `Ctrl + Shift + R`.

---

## 3. Telemetry: Event Monitor vs Sysmon vs Wazuh

Event Monitor, Sysmon, and Wazuh are different tools for monitoring and analyzing system events and security incidents. Event Monitor can only show the events that are logged by the system, while Sysmon is a more advanced tool that can provide detailed information about system activity, including process creation, network connections, and file modifications. Wazuh, on the other hand, is a comprehensive security monitoring platform that can collect and analyze data from multiple sources, including event logs, Sysmon data, and other security tools. It provides a centralized view of security events and can generate alerts based on predefined rules and policies.

Telemetry refers to the process of collecting and transmitting data from remote sources to a central system for monitoring and analysis. In the context of Sysmon, it gives more telemetry data than Event Monitor, which can be useful for detecting and investigating security incidents. The same applies to auditd on Linux, which provides detailed auditing capabilities for monitoring system activity and security events. By collecting telemetry data from various sources, security teams can gain insights into potential threats and take proactive measures to protect their systems and networks.

Sysmon works with an XML file for configuration. There are two common types of config (though you can still customize it): SwiftOnSecurity and Olaf Hartong's sysmon-modular. A Sysmon config file works by whitelisting or blacklisting certain events, processes, or behaviors based on the defined rules. The SwiftOnSecurity config is a widely used configuration that provides a good balance between security and usability, while Wazuh's own config is tailored to work seamlessly with the Wazuh platform, allowing for better integration and detection capabilities. Users can customize these configs to suit their specific security needs and environment. Auditd is similar to Sysmon in that it provides detailed auditing capabilities for monitoring system activity and security events on Linux systems. It can be configured to log specific events, processes, or behaviors, allowing for granular control over what is monitored and reported. Both Sysmon and auditd are valuable tools for enhancing security monitoring and incident response efforts.

`ossec.conf` is the main configuration file for Wazuh, which defines various settings and parameters for the Wazuh manager and agents. We can find this file in `Program Files (x86)/ossec-agent`. It includes configurations for log collection, alerting, rules, decoders, and other components of the Wazuh platform. The `ossec.conf` file allows administrators to customize the behavior of Wazuh to meet their specific security monitoring requirements. It is important to carefully configure this file to ensure that the Wazuh system operates effectively and efficiently in detecting and responding to security threats. For Sysmon, I need to set the config file so that Wazuh can see the events that Sysmon has logged. `sysmon64` is the executable used to install Sysmon on 64-bit Windows. After that, we can modify the `ossec.conf`. This is done on the Wazuh agent, so on my Windows 10 VM.

I can use `Ctrl + Shift + Enter` to launch an app in Administrator mode, which is useful for performing tasks that require elevated privileges, such as installing software, modifying system settings, or running commands that affect the entire system. This shortcut provides a quick and convenient way to launch applications with administrative rights without having to navigate through menus or use the right-click context menu.

---

## 4. Custom Detection Rules

Next, I tried detecting the DeerStealer malware with Wazuh. I can do this by creating a new file, but I can also access it via the GUI from the Wazuh dashboard by going to Server Management > Create New Rule. Here's how the rule syntax works:

### Rule Syntax

- The `rule` tag has a unique ID number (if you make it within the range 100000 to 120000), and the level shows how serious it is. 0 is for ignore, 3-6 is low/medium, 7-11 is high, and 12 and above is critical.
- `if_sid` means "only fire this rule if the parent rule with this ID has fired." This is useful for creating hierarchical rules where certain conditions must be met before a rule is triggered. So, it acts as a chained condition.
- The `field` tag means matching a field in the log. For example, if I want to match the field "srcip" in the log, I can use the field tag with `name="srcip"` and then set the value to be matched. Usually, it's used with pcre2 or regex for the matching.
- `description` shows the human-readable text displayed in the alert if the regex matches.
- `mitre` maps the alert to which MITRE ATT&CK technique it belongs to. This will show the technique ID and name.
- `group` is used to categorize the rule into a specific group or category, which can help in organizing and managing rules within the Wazuh platform. We use commas to separate multiple groups if needed.

### DeerStealer Malware

AnyRun is needed so I can download the malware sample, which is the DeerStealer malware. AnyRun is an interactive online malware analysis sandbox that allows users to safely analyze and observe the behavior of malicious files in a controlled environment. By using AnyRun, I can obtain the necessary malware samples for testing and detection purposes without risking the security of my own systems. DeerStealer is a malware that obtains passwords from web browsers and steals cryptocurrency, then sends it to the attacker. It's not a worm, so it doesn't spread across the network and only affects the host where the executable is run. However, it might exploit a vulnerability in VirtualBox to execute on the host laptop, especially if there are any shared resources. A VM isn't always safe or isolated if the malware can spread to other devices on a network.

---

## 5. Detecting cipher.exe

Instead of running DeerStealer, I tried to create an alert for running `cipher.exe`. `cipher.exe` is a legitimate Windows utility used for managing encryption on NTFS volumes. However, it can also be exploited by attackers to perform malicious activities, such as encrypting files for ransomware attacks. By creating an alert for the execution of `cipher.exe`, I can monitor and detect any suspicious usage of this utility, which may indicate potential security threats or unauthorized actions on the system. In the process of creating the rule, I realized that process creation (Event ID 1) was not included in the Sysmon config whitelist, so I needed to add it manually. Previously, only Event ID 10 was being logged for `cipher.exe`, where PowerShell called `cipher.exe`. After fixing this, I can now see the alert for `cipher.exe` from Event ID 1, which is the process creation event. Monitoring for Event ID 1 is better than Event ID 10 because Event ID 1 is the first event that occurs when a process is created, while Event ID 10 is generated later in the process lifecycle. By monitoring for Event ID 1, I can detect the execution of `cipher.exe` as soon as it starts, allowing for quicker response and mitigation of potential threats. If I only monitor Event ID 10, I might miss the initial execution of `cipher.exe`, which could delay detection and response to malicious activity. Also, there might be multiple Event ID 10 entries for the same process, which can lead to confusion and make it harder to identify the actual threat.

I use `.\sysmon64.exe -c sysmonconfig.xml` to update the Sysmon config. The `-c` option specifies the configuration file to be used, allowing me to apply the new settings defined in `sysmonconfig.xml`. This command updates the existing Sysmon configuration with the rules and parameters specified in the provided XML file.

---

## 6. File Integrity Monitoring (FIM)

Next, I learned about FIM (File Integrity Monitoring). **I DID THIS ON MY UBUNTU VM.** Basically, it checks if a file has been modified by tracking the checksums of the files. If the checksum changes, it means the file has been modified. This is useful for detecting unauthorized changes to critical system files, configuration files, or other important data. FIM can help identify potential security breaches, malware infections, or accidental modifications that could compromise the integrity of the system. By monitoring file integrity, organizations can ensure that their systems remain secure and compliant with security policies and regulations.

To do this, I need to modify the `agent.conf` file in Wazuh. I used this on the Linux OS and configured `check_all` and `realtime` to `yes`. Realtime means the alert will come immediately after the file is modified, while `check_all` means it will check every attribute of the files, like `check_sum`, `check_size`, `check_owner`, `check_perm`, etc. This ensures that any changes to the monitored files are detected and reported promptly, allowing for timely investigation and response to potential security incidents. For the path, it needs to be an absolute path; to get the value of `~`, I need to run `echo $HOME`.

---

## 7. VirusTotal Integration

For the VirusTotal integration, I need to use the VirusTotal API key and add it to `ossec.conf` on the Wazuh server (with Docker, the config file is in the Docker config folder, not in the `/var` folder of the host). The key will be added to the `ossec.conf` file in the manager. This means if there are file changes in the FIM, the Wazuh manager will send the file to VirusTotal for scanning. If VirusTotal returns a positive result, the Wazuh manager will generate an alert. This integration allows for automated malware detection and enhances the overall security monitoring capabilities of the Wazuh platform. Because I used Docker, the `wazuh_manager.conf` file needs to be changed, and then `docker-compose down` and `up` again to apply the changes. There's no need to run `systemctl restart wazuh-manager` because `docker-compose down` and `up` will restart the Wazuh manager container.

---

## 8. Active Response

To enable active response, I need to install the `jq` library on the Wazuh agent. The `jq` library is used to process JSON data from Wazuh and then implement automated response actions. I also realized that it's better to SSH into the Ubuntu server VM rather than installing Guest Additions (which I don't know how to do manually). Here are the steps to enable SSH on the VM:

### SSH Setup for Ubuntu VM

```
sudo apt install -y openssh-server
sudo systemctl enable --now ssh
sudo ss -tlnp | grep :22
ip -4 addr show | grep inet
```

### VirtualBox Network Modes

The VM must use a bridged adapter, not NAT, so that the VM can be accessed from the host machine. The different network modes in VirtualBox are:

- **NAT (Network Address Translation):** The VM shares the host's IP address and can access external networks, but it is not directly accessible from the host or other devices on the network.
- **NAT network:** Similar to NAT, but allows multiple VMs to communicate with each other within a private network while still sharing the host's IP address for external access.
- **Bridged adapter:** The VM is connected directly to the physical network, allowing it to have its own IP address and be accessible from other devices on the same network, including the host machine.
- **Internal network:** The VM can communicate only with other VMs on the same internal network, but it cannot access external networks or the host machine.
- **Host-only adapter:** The VM can communicate only with the host machine and other VMs configured with the same host-only network, but it cannot access external networks.
- **Generic driver:** This option allows for the use of third-party network drivers, which may provide additional features or compatibility with specific network configurations.

### remove-threat Script

Also edit the Docker Wazuh manager config file again:

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

`chown` is used to change the ownership of a file. In this case, I need to change the ownership of `remove-threat.sh` to the root user and root group. This is important for security reasons, as it ensures that only authorized users have access to the script and can execute it. By setting the correct ownership, I can prevent unauthorized modifications or execution of the script, which could potentially compromise the security of the system.

Setting `chmod` to 750 for the `remove-threat.sh` file sets the file permissions, allowing the owner (root) to read, write, and execute the script, while the group (root) can read and execute it, and others have no permissions. This permission setting ensures that only authorized users can access and execute the script, enhancing the security of the system. Aside from 750, there is also 755, which allows the owner to read, write, and execute the script, while the group and others can read and execute it.

---

## 9. MariaDB Auditing

Still on the Ubuntu VM, I wanted to install MariaDB. Because I'm using Ubuntu Server instead of Fedora (as stated in the documentation), I had to tweak a few things.

### Installation (Ubuntu 22.04)

```
sudo apt install mariadb-server mariadb-client galera-4
sudo mariadb-secure-installation
sudo systemctl status mariadb
sudo mariadb -u root
```

### Audit Configuration

After that, we can add the Wazuh config to MariaDB (this is the part where it differs from the docs):

```
sudo nano /etc/mysql/mariadb.conf.d/50-audit.cnf
<copy paste the content of 50-audit.cnf from my directory>
sudo systemctl restart mariadb
sudo tail -f /var/log/syslog | grep mariadb-server_auditing (check if it's working)
```

### Decoders & Rules

Wazuh has thousands of default rules. If there were no decoders, every time a single log arrived, the manager would have to run complex regex operations against every single rule to see if it matched. Your CPU would max out at 100% almost immediately, and log processing would grind to a halt.

- **Without Decoders:** You would have to write complex PCRE2 regex in your rule to account for the Windows format, the Linux format, the Cisco format, etc. You would need dozens of duplicate rules.
- **With Decoders:** The Windows decoder, the Linux decoder, and the Cisco decoder all translate their unique logs into the exact same variable: `srcip`. Now, you only have to write one single rule that says: "If `srcip` fails 5 times, trigger an alert."

After inserting the MariaDB decoders and rules into the Wazuh manager, I used the GUI instead of manually inserting them into `/var/ossec/etc/rules` and `/var/ossec/etc/decoders`.

Now that it's set up, I can monitor multiple activities from MariaDB, such as authentication operations, data definition operations, data manipulation operations, data destruction operations, data access and query operations, and account and privilege operations.

---

## 10. MITRE ATT&CK Framework

### 1. The Headers = Tactics (The Attacker's Goal)

The big bold headers at the top (like Initial Access, Execution, Persistence) are called **Tactics**.

Think of these as the attacker's high-level objectives. They answer the question: "What is the hacker trying to achieve at this exact moment?"

For example, under Initial Access, the hacker's goal is simply to get a foot in the door of your network. Under Persistence, their goal is to make sure they don't lose access if a server reboots.

### 2. The Cells Below = Techniques (The Attacker's Method)

The individual boxes underneath each header (like Drive-by Compromise, BITS Jobs, Command and Scripting Interpreter) are called **Techniques**.

Think of these as the specific actions or methods the hacker uses to accomplish that top header goal. They answer the question: "How exactly is the hacker pulling this off?"

For example, if the attacker's goal is Execution (running malicious code on your server), one technique they might use is a BITS Job or a Command and Scripting Interpreter (like running a malicious PowerShell or Bash script).

### Where does the "Defense" come in?

The Navigator starts completely grey because it only lists the hacker's playbook. The defense part is where you have to do it yourself (either manually or automatically). You color-code it: red for the techniques that you have detected, yellow for the techniques that you have partially detected, and green for the techniques that you have fully detected. This allows you to visualize your defensive coverage against the attacker's methods and identify areas where you may need to improve your security posture. By mapping your detections to the MITRE ATT&CK framework, you can better understand the tactics and techniques used by attackers and enhance your overall threat detection and response capabilities.

One thing to note is that, for example, BITS Jobs is a technique that comes under the tactics Execution, Persistence, and Defense Evasion. This means that if I color-code BITS Jobs, it will be colored in all three tactics. This is because the same technique can be used by attackers to achieve multiple objectives, and by mapping it to multiple tactics, I can gain a more comprehensive understanding of the attacker's methods and potential impact on my systems.

---

## 11. SBOM & Dependency-Track

An SBOM can be generated for web apps or mobile apps. SBOM stands for Software Bill of Materials, which is a comprehensive list of all the components, libraries, and dependencies used in a software application. It provides transparency into the software supply chain and helps organizations identify potential security vulnerabilities or licensing issues associated with the components used in their applications. By generating an SBOM, developers and security teams can better understand the composition of their software and take proactive measures to mitigate risks and ensure compliance with security standards and regulations.

```
npm install --save-dev @cyclonedx/cyclonedx-npm
```

`--save-dev` is used for development dependencies.

Now, command the tool to scan your project and spit out a JSON-formatted bill of materials:

```
npx @cyclonedx/cyclonedx-npm --output-format JSON --output-file bom.json
```

Or if using Python, then:

```
pip install cyclonedx-bom
cyclonedx-py -o backend-bom.json
```

An example of a BOM is `game-centr-frontend-bom.json`. This file is from the frontend of my Game Centr project.

After that, the Wazuh way is to upload that SBOM to Dependency-Track and use Wazuh to generate the alerts. Dependency-Track is an open-source software composition analysis (SCA) platform that helps organizations identify and manage security vulnerabilities in their software dependencies. By uploading the SBOM to Dependency-Track, I can analyze the components used in my project and receive alerts for any known vulnerabilities or security issues associated with those components. Wazuh can then be configured to monitor the Dependency-Track platform and generate alerts based on the findings, allowing for proactive management of software security risks. This integration enhances the overall security posture of the application by providing visibility into potential threats and enabling timely remediation actions.

---

## 12. Wazuh OVA

There is also Wazuh OVA (Open Virtual Appliance). This Wazuh image is already pre-installed with the Wazuh dashboard, Wazuh manager, and Wazuh indexer. This is useful for quickly setting up a Wazuh environment without having to go through the manual installation and configuration process. The OVA can be imported into virtualization platforms like VMware or VirtualBox, allowing for easy deployment and testing of Wazuh in a controlled environment. It provides a convenient way to evaluate the capabilities of Wazuh and its components without the need for extensive setup or configuration.

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

Wazuh is an open-source SIEM/XDR platform that monitors, detects, and responds to security threats across endpoints, networks, and cloud environments in real time. Zeek, on the other hand, is a Linux-based open-source framework for network visibility that analyzes traffic and produces structured logs and artifacts describing observed activity. It's for network monitoring / network telemetry; it mostly observes and records. By ingesting Zeek logs, Wazuh enriches them with security context and applies correlation, alerting, and automated response actions. Integrating Zeek with Wazuh delivers unified, actionable visibility into network events and strengthens an organization's ability to detect, investigate, and respond to security threats.

The documentation for installing Zeek uses Ubuntu 24. Because I'm using Ubuntu 22, I needed to tweak a few things:

```
curl -fsSL https://download.opensuse.org/repositories/security:zeek/xUbuntu_22.04/Release.key | \
gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/security_zeek.gpg > /dev/null
echo 'deb http://download.opensuse.org/repositories/security:/zeek/xUbuntu_22.04/ /' | \
sudo tee /etc/apt/sources.list.d/security:zeek.list
sudo apt update
sudo apt install zeek -y
```

Another error I found was that `zeekctl` was not in my system PATH, so I had to use the full path:

```
/opt/zeek/bin/zeekctl
```

I also fixed the network interface (MAIN FIX). I changed Zeek from using (this is because `eth0` is not found when I run the `ip a` command):

```
eth0
```

to:

```
enp0s3
```

Note: sometimes the password for the Wazuh OVA is actually `admin`, not `SecretPassword`.

---

## 15. FIM Configuration Note

FIM is configured through the agent or centralized syscheck configuration (`agent.conf`), while the Endpoint Security FIM tab only displays the alerts generated by the syscheck engine after the agent has applied those rules and detected file changes.
