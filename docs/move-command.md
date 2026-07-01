# Moving Files from VPS to Local Machine

To move files from VPS to local machine, I use the `scp` command. There is the `-r` flag for recursive copy, which is useful for copying directories.

```bash
scp -r root@VPS_IP:~/wazuh-docker ./wazuh-docker
```
