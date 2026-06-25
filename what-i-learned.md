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

to move files from vps to local machine, i use scp command. There is the flag -r for recursive copy, which is useful for copying directories.
