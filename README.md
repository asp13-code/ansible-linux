# INSTALL WIREGUARD

This Ansible role installs and configures and starts wireguard on inventory linux hosts in a hub-n-spoke (aka star) topology where one of the hosts considered a server and other hosts clients.

Role is semi-idempotent, i.e. multiple runs won't change wireguard configuration, but service will be restarted to apply possible config changes.


## How to set hosts and variables
All hosts and wireguards specific configuration should be set in inventory host to maintain consistent view of IP addresses used in VPN overlay network.

### Example of inventory file
```ini
[general_linux_group]
wg-srv01 ansible_host=192.168.1.11
wg-client02 ansible_host=192.168.2.12
wg-client03 ansible_host=192.168.100.13
wg-client02 ansible_host=192.168.128.14
wg-client05 ansible_host=192.0.2.15


[wireguard]
wg-srv01 wgrole="server" wgip="10.10.10.1" wgsrvip="192.168.1.11" wgsrvport="51820" wgdnsip="8.8.8.8" wgkeepalive="30" wgroute="10.10.10.0/24"
wg-client02 wgrole="client" wgip="10.10.10.2"
wg-client03 wgrole="client" wgip="10.10.10.3"
wg-client04 wgrole="client" wgip="10.10.10.4"
wg-client05 wgrole="client" wgip="10.10.10.5"
```

### Variables description
`wgrole` could be `server` or `client` where server role specified for one host only

`wgip` is host internal IP in VPN overlay network

`wgsrvip` considered as external IP of a server aka 'IP to connect to'. Could be the same as specified in ansible inventory linux group or any other including internet/public IP.

`wgsrvport` server port to connect to. 51820 is default wireguard value.

`wgdnsip` DNS server to use. Will become the main host resolver unless link/domain-specific settings are implemented with systemd/resolved and/or wireguard services stopped. `wgdnsip` could be empty.

`wgkeepalive` is heartbeat to detect stale connections

`wgroute` is a route that specifies what traffic to redirect to VPN tunnel

## How to run
```bash
ansible-playbook playbooks/wireguard.yml
```
Role could be called from a playbook using varios switches (turning on and off functionality)

### Example of a playbook
```yaml
---
- name: Installing, enabling boot and starting wireguard
  hosts: wireguard
  gather_facts: false
  become: true
  tasks:

  - name: calling a role
    include_role:
      name: software-install-wireguard
    vars:
      wgzeroize: false
      wgconfigure: true
      wgenable: true
      wgstart: true
```

### Variables description
There are various switches that could be used to control role behavior

`wgzeroize` will delete public and private keys, wg.conf file. Use wisely to configure wireguard from the scratch. Will be executed before configuration code

`wgconfigure` main code that generates wireguard keys (if needed) and wg.conf content

`wgenable` whether to enable wireguard service at boot time

`wgstart` whether to start/restart wireguard service

If all variables are false or absent role just installs wireguard package.

### Managing wireguard installation
Additionally, you can run parent wireguard playbook with tags to manage wireguard installation
```bash
ansible-playbook playbooks/wireguard.yml --tags=<action>
```
Action is one of the following: `enable`, `start`, `restart`, `stop`, `disable`

## Performance
Here is some unverified/unexplained reference digits that were gotten on a Intel(R) Xeon(R) CPU E5-2667 @ 2.90GHz. Ubuntu 20.04 qemu images on Ubuntu 18.04 host machine. Wireguard server and clients were on the same machine.

Measured with iperf v2

iPerf Server
```bash
iperf -s -f m -i 1
```

Client
```bash
iperf -c 10.10.10.1 -t 30
```

### Unencrypted performance client <-> server
Client <-> Server: 14 Gbits/sec

### Wireguard encrypted performance
Client <-> Server:            2 Gbits/sec

Client <-> Client:       900 Mbits/sec

## Network and firewall considerations

Nftables/iptables/ufw configuration is out of scope of this role.

Wireguard server binds UDP 51820 (default). Wireguard clients bind UDP dynamic port (high ranges)

Role enables routing for server machine in sysctl.conf.

Typical recovery times in a test setup were:
 * after wireguard server reboot 60-120 seconds
 * after service restart - 10-20 seconds.

Out-of-band connection (not affected by wireguard) is recommended from ansible control node to wireguard hosts

## Dependencies
Role tested on and meant to be executed on Ubuntu Linux 20.04

Role installs following apt packages:
```bash
wireguard
wireguard-tools
resolvconf
```
