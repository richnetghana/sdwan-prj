# sdwan-prj


# 📘 SD-WAN Deployment on OPNsense Using Docker and Ansible

## 📌 Overview

This document provides a step-by-step guide to deploying a **containerized SD-WAN solution on OPNsense**, orchestrated with **Ansible** for automation. The deployment leverages Docker for lightweight virtualized services and OPNsense for routing, firewall, and VPN functionality.

## ⚙️ Prerequisites

- A server or VM running **OPNsense (v23 or higher)** with:
  - Access to the OPNsense Web GUI & SSH
  - WAN and LAN interfaces configured
- A **Docker host** (Linux VM or bare metal) reachable from OPNsense
- **Ansible** installed on your control machine
- Basic familiarity with:
  - Networking (IP, routes, VPNs)
  - Docker & Compose
  - YAML syntax
  - Firewall rules and NAT

## 🧱 SD-WAN Architecture Overview

```
     [Remote Branch]        [Docker Host]           [OPNsense Router]        [Internet]
         +------+            +----------+            +--------------+         +-----+
         | VPN  | <--------> | SD-WAN   | <--------> | Firewall     | <-----> | ISP |
         | Node |            | Gateway  |            | & Router     |         +-----+
         +------+            +----------+            +--------------+
                                (Docker)
```

## 📁 Folder Structure

```bash
sdwan-opnsense/
├── ansible/
│   ├── inventory.ini
│   ├── playbook.yml
│   └── roles/
│       ├── docker-host/
│       └── opnsense/
├── docker/
│   ├── docker-compose.yml
│   └── configs/
│       └── sdwan.conf
└── README.md
```

## 🚀 Step 1: Prepare the Docker Host

Install Docker and Docker Compose on the host.

```bash
sudo apt update
sudo apt install -y docker.io docker-compose
sudo systemctl enable docker --now
```

### Example `docker-compose.yml`

```yaml
version: "3.8"
services:
  sdwan-node:
    image: zerotier/zerotier
    container_name: sdwan-node
    network_mode: "host"
    cap_add:
      - NET_ADMIN
      - SYS_ADMIN
    volumes:
      - /var/lib/zerotier-one:/var/lib/zerotier-one
    restart: unless-stopped
```

## 🔧 Step 2: Set Up OPNsense

### Enable SSH Access
- Go to **System > Settings > Administration**
- Enable **SSH** and set up an **admin user** with shell access.

### Install Plugins (Via Web UI or SSH)
Install these essential plugins:
- `os-wireguard`
- `os-openvpn`
- `os-dyndns`
- `os-firewall`

### Configure Interfaces
- Assign virtual interfaces for VPN tunnels (e.g., `wg0`, `tun0`)
- Configure LAN/WAN appropriately
- Enable DHCP or static addressing

## 🤖 Step 3: Automate with Ansible

### 3.1 Inventory File (`inventory.ini`)

```ini
[docker]
docker-host ansible_host=192.168.1.100 ansible_user=ubuntu

[opnsense]
opnsense-router ansible_host=192.168.1.1 ansible_user=admin ansible_password=yourpassword ansible_connection=ssh
```

### 3.2 Ansible Playbook (`playbook.yml`)

```yaml
- name: Deploy SDWAN Node to Docker Host
  hosts: docker
  become: yes
  tasks:
    - name: Copy docker-compose file
      copy:
        src: ../docker/docker-compose.yml
        dest: /home/ubuntu/sdwan/docker-compose.yml

    - name: Deploy containers
      command: docker-compose -f /home/ubuntu/sdwan/docker-compose.yml up -d

- name: Configure OPNsense
  hosts: opnsense
  tasks:
    - name: Enable WireGuard plugin
      uri:
        url: "https://{{ inventory_hostname }}/api/wireguard/client/start"
        method: POST
        user: "{{ ansible_user }}"
        password: "{{ ansible_password }}"
        force_basic_auth: yes
        validate_certs: no
```

> 🔐 *For production, use OPNsense API Keys and Ansible Vault for credentials.*

## 🌐 Step 4: Establish VPN Links

You can use **WireGuard**, **OpenVPN**, or **ZeroTier** for SD-WAN links.

### Example: WireGuard Setup

- Configure `wg0` on both ends (Docker container & OPNsense)
- Exchange public keys and set allowed IPs
- Enable NAT & routing

## 🛡️ Step 5: Set Firewall Rules in OPNsense

- Go to **Firewall > Rules > WAN**
  - Allow WireGuard port (e.g., UDP 51820)
- Go to **Firewall > NAT > Outbound**
  - Add rules to allow routing from the VPN network to the WAN

## 🧪 Step 6: Test the Setup

- From the Docker container, ping the LAN behind OPNsense
- Test site-to-site communication between Docker-based SD-WAN nodes
- Confirm route propagation and failover behavior

## 🧰 Optional Enhancements

- Integrate with **Zabbix**, **NetBox**, or **Grafana** for monitoring
- Use **WireGuard tunnels over multiple ISPs** for failover
- Use **HAProxy or FRR** for dynamic routing on OPNsense
- Add **ZeroTier Moon** controller for private SD-WAN mesh

## 🧼 Troubleshooting Tips

- Use `tcpdump` and `ping` on OPNsense to verify traffic
- Use `docker logs` to debug container services
- Ensure ports are open on both the Docker host and OPNsense
- Check OPNsense's firewall logs for blocked traffic

## 📚 References

- [OPNsense Docs](https://docs.opnsense.org/)
- [WireGuard](https://www.wireguard.com/)
- [Ansible Docs](https://docs.ansible.com/)
- [ZeroTier](https://www.zerotier.com/)
- [Docker Compose](https://docs.docker.com/compose/)

## 🏁 Conclusion

This guide provides a flexible, automation-friendly SD-WAN deployment using Docker and OPNsense. It’s scalable across multiple branches and easily manageable using Ansible.

For enterprise-grade deployments, consider integrating with orchestration platforms like **Terraform**, **NetBox**, or **SaltStack**.
