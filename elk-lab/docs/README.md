# ELK Lab — Phase 0 (Vagrant, Debian Bullseye)
- Provider: VirtualBox
- Network: Private `192.168.57.0/24`
- Nodes: lb, web1, web2, db, elk

## Commands
```bash
vagrant up --no-provision
vagrant status
vagrant ssh lb     # یا هر نود دیگر
vagrant ssh lb -c "cat /etc/os-release"
