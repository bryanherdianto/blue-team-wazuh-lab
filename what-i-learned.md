So, i used docker deployment instead of manually deploying wazuh manager, wazuh indexer, and wazuh dashboard. this is because independent download may need independent dependencies and configuration that can be cumbersome. Docker deployment simplifies the process by encapsulating all necessary components and their dependencies into containers, making it easier to manage and deploy. Also, there are more steps to follow in the manual deployment, which can be time-consuming and error-prone, like initial config, node isntall, cluster init, and testing the cluster. Docker deployment, on the other hand, allows for a more streamlined setup process, reducing the potential for mistakes and saving time.

Also, the documents asked me to sysctl -w vm.max_map_count=262144... but this has a flaw in the sense that it is not persistent across reboots. To make it persistent, I had to add the line vm.max_map_count=262144 to /etc/sysctl.conf and then run sysctl -p to apply the changes. This ensures that the setting remains in effect even after a system restart. this is the full command:

```
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

Also, it's supposed to be docker-compose, not docker compose

alas, running docker-compose up -d is mandatory i guess, i mean like it's the default, you can omit the -d if you want to see the logs in the terminal, but if you want to run it in the background, -d is necessary.

generate-indexer-certs is used to generate TLS certificates for secure communication between the Wazuh components. This is important for ensuring that data transmitted between the manager, indexer, and dashboard is encrypted and secure. The command typically creates the necessary certificate files and places them in the appropriate directories for use by the Wazuh services.

i can use free -h to see current total ram and available ram. even though, the document says cpu needed is 4, but my current vps has spec 2vpcu and 8gb ram... and it still works, so for my small scale lab, it is sufficient. but for production, i guess 4vpcu and 8gb ram is the minimum requirement.

docker ps and docker-compose ps are different because docker ps shows all running containers on the system, while docker-compose ps shows the status of containers defined in the docker-compose.yml file for a specific project. This distinction is important for managing and monitoring containers in a multi-container environment.

you can see the public ip of your vps by using the command curl ifconfig.me. This command queries an external service to return the public IP address of your VPS, which can be useful for accessing services running on the VPS from outside networks.

Because docker set this 0.0.0.0:443->5601/tcp, then it means that by accessing the host's IP on port 443 or just HTTPS, you can reach the Wazuh dashboard. The mapping indicates that the container's port 5601 is exposed to the host's port 443, allowing secure access to the dashboard via HTTPS. The user and password for the wazuh dashboard is username is admin and password is SecretPassword.

For this lab, im gonna install wazuh agent on my 2 vms. One vm is windows, while the other is ubuntu. The Wazuh agent is responsible for collecting and sending security data from the endpoints to the Wazuh manager for analysis. Installing the agent on both Windows and Ubuntu VMs allows for monitoring and securing different operating systems within the lab environment. Later, i can install vulnerabilities or malware in the VMs to test the detection capabilities of the Wazuh setup. This will help in evaluating how well the Wazuh system can identify and respond to security threats across different platforms.

To deploy agent in the windows:

1. Set package to download to be Windows the MSI 32/64 bits.
2. Set the server address to be the public ip of the vps where the wazuh manager is running.
3. SEt agent name into windows10 (since it's OS is Windows 10)
4. After that, just copy paste the commands that Wazuh has provided on Powershell with Administraotyr privileges
5. Do NET START Wazuh
6. If they already show up, then good. if not, then Do hard refresh on wazuh dashboard by Ctrl + Shift + R.

TO deploy the agent in the ubuntu:

1. Set package to download to be Linux the DEB and AMD64 architecture. For reference, DEB includes ubuntu, debian, and other debian-based distributions. Meanwhile, RPM includes centos, fedora, and other redhat-based distributions.
2. Set the server address to be the public ip of the vps where the wazuh manager is running.
3. Set agent name into ubuntu22-server (since it's OS is Ubuntu 22.04 server)
4. After that, just copy paste the commands that Wazuh has provided on terminal with root privileges (used sudo in front of all the commands)
5. Again with sudo, do systemctl daemon-reload, systemctl enable wazuh-agent, and systemctl start wazuh-agent.
6. If they already show up, then good. if not, then Do hard refresh on wazuh dashboard by Ctrl + Shift + R.

Event monitor, sysmon, and wazuh are different tools for monitoring and analyzing system events and security incidents. Event monitor can only show the events that are logged by the system, while sysmon is a more advanced tool that can provide detailed information about system activity, including process creation, network connections, and file modifications. Wazuh, on the other hand, is a comprehensive security monitoring platform that can collect and analyze data from multiple sources, including event logs, sysmon data, and other security tools. It provides a centralized view of security events and can generate alerts based on predefined rules and policies.

Telemetry means the process of collecting and transmitting data from remote sources to a central system for monitoring and analysis. In the context of sysmon, it gives more telemetry data than event monitor, which can be useful for detecting and investigating security incidents. This is also the same for auditd in Linux, which provides detailed auditing capabilities for monitoring system activity and security events. By collecting telemetry data from various sources, security teams can gain insights into potential threats and take proactive measures to protect their systems and networks.

Sysmon works with XML file to config. There is 2 common types of config (but still doesnt change the fact you can customize it), that is by using SwiftOnSecurity and Olaf Hartong's sysmon-modular. Sysmon config file works by whitelisting or blacklisting certain events, processes, or behaviors based on the defined rules. The SwiftOnSecurity config is a widely used configuration that provides a good balance between security and usability, while Wazuh's own config is tailored to work seamlessly with the Wazuh platform, allowing for better integration and detection capabilities. Users can customize these configs to suit their specific security needs and environment. Auditd is similar to sysmon in that it provides detailed auditing capabilities for monitoring system activity and security events on Linux systems. It can be configured to log specific events, processes, or behaviors, allowing for granular control over what is monitored and reported. Both sysmon and auditd are valuable tools for enhancing security monitoring and incident response efforts.

ossec.conf is the main configuration file for Wazuh, which defines various settings and parameters for the Wazuh manager and agents. We can find this file in Program Files (x86)/ossec-agent. It includes configurations for log collection, alerting, rules, decoders, and other components of the Wazuh platform. The ossec.conf file allows administrators to customize the behavior of Wazuh to meet their specific security monitoring requirements. It is important to carefully configure this file to ensure that the Wazuh system operates effectively and efficiently in detecting and responding to security threats. Like for sysmon, i need to set the config file so that wazuh can see the events that sysmon has logged. AFter using, sysmon64 is the exe used to install sysmon on windows with 64 bit architecture. Then after that, we can change the ossec.conf. BTW this is done on Wazuh agent, so on my VM windows 10.

I can use Ctrl + Shift + enter to enter the app in administrator mode, which is useful for performing tasks that require elevated privileges, such as installing software, modifying system settings, or running commands that affect the entire system. This shortcut provides a quick and convenient way to launch applications with administrative rights without having to navigate through menus or use the right-click context menu.

Right next, im trying out detecting deerstealer malware with wazuh. I can do it by creating the new file, but i can also access it via GUI from the Wazuh dashboard > going to the server management > create new rule. Here's how the rule syntax works:

- rule tag has a unique id number (if you make make it within the range 100000 to 120000) and the level shows how serious it is. Like, 0 is for ignore, 3-6 to low/medium, and 7-11 is high, while 12 above is critical.
- if_sid is like only fire this rule if the parent rule with id if_sid has fired. This is useful for creating hierarchical rules where certain conditions must be met before a rule is triggered. So, its like a chained condition.
- field tag means matching the field in the log. For example, if i want to match the field "srcip" in the log, then i can use field tag with name="srcip" and then set the value to be matched. Usually, it's used with pcre2 or regex for the matching.
- description shows the human text shown in alert if the regex matches.
- mitre maps the alert to which mitre attack technique it belongs to. This will show the technique ID and name.
- group is used to categorize the rule into a specific group or category, which can help in organizing and managing rules within the Wazuh platform. We use comma to separate multiple groups if needed.

AnyRun is needed so i can download the malware sample, which is deerstealer malware. AnyRun is an interactive online malware analysis sandbox that allows users to safely analyze and observe the behavior of malicious files in a controlled environment. By using AnyRun, I can obtain the necessary malware samples for testing and detection purposes without risking the security of my own systems. So, Deerstealer malware is a malware that obtains password from web browsers and take crypto money, then sent it to the attacker. It's not a worm, so it doesn't spread to the network and only effects the host where the exe is run. But, it might use the vulnerability in virtualbox to execute in host laptop, and if there are any shared resources. Btw, VM isn't always safe or isolated if the malware can spread to other devices in a network.

So, instead of running deerstealer, i tried to make alert for running cipher.exe. Cipher.exe is a legitimate Windows utility used for managing encryption on NTFS volumes. However, it can also be exploited by attackers to perform malicious activities, such as encrypting files for ransomware attacks. By creating an alert for the execution of cipher.exe, I can monitor and detect any suspicious usage of this utility, which may indicate potential security threats or unauthorized actions on the system. In the process of creating the rule, I realize that process creation or event ID 1 is not being included in the whitelist of sysmon config, so i need to add it in manually. Previously, only event ID 10 is being logged for cipher.exe in which powershell is calling cipher.exe. AFter fixing this, now i can see the alert for cipher.exe from event ID 1, which is the process creation event. Why monitoring for event ID 1 is better than event ID 10 is because event ID 1 is the first event that occurs when a process is created, while event ID 10 is generated later in the process lifecycle. By monitoring for event ID 1, I can detect the execution of cipher.exe as soon as it starts, allowing for quicker response and mitigation of potential threats. Meanwhile, if i only monitor event ID 10, i might miss the initial execution of cipher.exe, which could delay detection and response to malicious activity. Also, there might be multiple event ID 10 for the same process, which can lead to confusion and make it harder to identify the actual threat.

i use .\sysmon64.exe -c sysmoncconfig.xml to update the sysmon config. The -c option specifies the configuration file to be used, allowing me to apply the new settings defined in sysmonconfig.xml. This command updates the existing sysmon configuration with the rules and parameters specified in the provided XML file.
