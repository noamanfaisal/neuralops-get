# NeuralOps Nexus — installer

Install NeuralOps Nexus on a Linux server with one command:

```
curl -fsSL https://raw.githubusercontent.com/noamanfaisal/neuralops-get/main/install.sh | bash
```

The installer asks one question:

1. **This server has a public IP** (AWS, Azure, Google, DigitalOcean, any cloud
   server) — it gets a free https address. Ports 80 and 443 must be open.
2. **Tailscale** — for a machine at home or in the office.

Needs: Linux (x86_64), 4 GB of memory, 15 GB of free disk. Docker is installed
for you if it is missing.

Afterwards, the `neuralops` command manages the server:
`neuralops status | address | update | backup | restore | logs | restart | stop | start | uninstall`

This repository holds only the installer files. NeuralOps itself ships as the
Docker image `noamanfaisal/neuralops`.
