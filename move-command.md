to move files from vps to local machine, i use scp command. There is the flag -r for recursive copy, which is useful for copying directories.

scp -r root@VPS_IP:~/wazuh-docker ./wazuh-docker