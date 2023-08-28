# pi-hole2vpn

## Why

1. I found only pre-built droplets on DigitalOcean and a lot of instructions about setting up Pi-hole and WireGuard, but I do not want to configure
    everything each time with so many settings. Ansible is an easy method to write some "setup notes" one time, and DigitalOcean has traffic limits
    on droplets.
2. When I use ad-blocker DNS, my phone stays cool and works for a full day (now it works 24 hours without charging, i finally measured it) :) It is truly the best method to increase battery lifetime.
3. I dislike spyware.

Look at screenshot. 80% requests it is spyware!
<p align="center">
  <img src="https://github.com/3DRaven/pi-hole2vpn/blob/master/images/Pi-hole-on-phone.png" width="150" height="280">
</p>

## How to use

Prerequirements:

1. Remote host with ssh access (tested DigitalOcean and Amazon VPS)
It is enough to have 512MB of RAM, 1 CPU core, and 5GB of disk space.
2. Ubintu 22.04 on remote host (tested only with Ubuntu 22.04)

Install steps:

0. Get EC2 instance on `aws.amazon.com` or Droplet on `digitalocean.com` or other VPS on any hoster
1. On LOCAL computer 
Install latest version of ansible
```bash
sudo apt-add-repository ppa:ansible/ansible
sudo apt-get update
sudo apt-get install ansible
```
2. Edit `group_vars/vpn.example`. It is file with main settings. And rename it to `vpn`
3. Edit `inventory.example` file to add IP of your remote hosts to install VPN+Pihole, in this file possible to set ssh access params.
And rename it to `inventory`
4. Execute command on LOCAL computer (in dir with deploy.yml file) 
```bash
ansible-playbook --ask-become-pass ./deploy.yml
```
If you do not need some actions just use tags. Available tags: [user_creation,vpn_installation,docker_installation,pi_hole_installation,adblock_add,adblock_remove,disable_ubuntu_user]. But it not tested.
example command:
```bash
ansible-playbook --ask-become-pass ./deploy.yml --tags adblock_add 
```    
5. insert REMOTE sudo password to prompt. At first run it is default for Ubuntu empty sudo password, next runs it is password from `group_vars/vpn->user_password`
6. After installation will be created dir `clients` in playbook dir. It is configuration files for clients and QR codes to scan from phone for connectiong to VPN.
7. At the end of instalation adblock lists from adlists_add.txt will be loaded to pihole and from adlists_remove.txt will be removed.
It is possible to run `adblock_add` `adblock_remove` tags separately if need at any time.
```bash
ansible-playbook ./deploy.yml --tags adblock_add,adblock_remove
```
8. At last step will be disabled login with `ubuntu` default user for Ubuntu. Next logins possible only with `group_vars/vpn->user_to_add` user name. So at first run `inventory` host description was
```
....3.eu-north-1.compute.amazonaws.com:22 ansible_ssh_user=ubuntu ansible_ssh_private_key_file=../../ubuntu.pem
``` 
at next runs after first success run it will be
```
....3.eu-north-1.compute.amazonaws.com:22 ansible_ssh_user={{ from  common_vars.yml->user_to_add }} ansible_ssh_private_key_file=../../key.pem
```
9. You do not need to doing something on remote host at all ;)
10. Playbook is not fully idempotent, but you can run it multiple times, but every time you will get new clients configs for connection to VPN. If you got any errors, just run it again. It is playbook for personal use, so we can just generate X configs for all our devices one time.

## State after playbook executed

1. Docker is installed on the remote host.
2. Pi-hole DNS is installed on the remote host.
3. All requests to port 53 inside the VPN will be redirected to the Pi-hole DNS, even if some spyware attempts to make a direct request to 8.8.8.8.
4. Zram is installed if `install_zram: true`. It is a good method to expand VPS RAM on the remote host. But you must have 
linux kernel with zram module. As example https://liquorix.net/#install
5. WireGuard is installed on the remote host.
6. Client configuration files are generated on the localhost. Will be generate two type of files: 
  a. Only DNS requests VPN. So, only DNS requests from client will be send to VPN, other traffic will be direct.
    This configs will be placed to ./clients/111-42.eu-north-1.compute.amazonaws.com/etc/wireguard/clients/wg0/dns
  b. All traffic over VPN. This in ./clients/111-42.eu-north-1.compute.amazonaws.com/etc/wireguard/clients/wg0/full
7. If default `group_vars/vpn->wireguard_listen_port` port is blocked all traffic from ports `group_vars/vpn->fallback_wireguard_listen_ports` will be redirected to `group_vars/vpn->wireguard_listen_port`
8. All unknown p2p TCP traffic not recognized by Pi-Hole to ports [123,137,443,80,8009] disabled and totaly all logged. 
Some spyware apps use direct requests. After I found this hidden traffic, battery lifetime significantly increased. 
Telegram use 5222,80,443 but 80,443 not only Telegram, so 80,443 were restricted and it works fine.
## Using VPN from phone:

1. Install wireguard client to phone
2. Scan QR code of any client from client dir (config_*.txt files it is QR codes) and connect to VPN
3. Open http://pi.hole/admin in browser (access only from VPN, password from `group_vars/vpn->pi_hole_admin_password`)

## Using from local computer

### Ubuntu

Command to install wireguard
```bash
sudo apt-get install wireguard
```
Command for import client configuration from file to NetworkManager: 
```bash
nmcli connection import type wireguard file ./client_4.conf
```
Command to connect: 
```bash
nmcli connection up client_4
```
Command to disconnect: 
```bash
nmcli connection down client_4
```
Command to delete connection:
```bash
nmcli connection delete client_4
```
## Troubleshooting

0. If you have problems with youtube just add `i.ytimg.com` `yt3.ggpht.com` to Pi-hole whitelist in http://pi.hole/admin
1. If you have problems with freezes tasks try to comment `inventory->ssh_connection` options. It is slower but may resolve some problems.
2. Do not forget to open ports 22 (SSH), 51820:51823 (default VPNs) on providers firewall
3. Sometimes the playbook can end up in an inconsistent state. For example, when systemd-resolved is stopped but the new DNS is not properly set, it might be necessary to recreate the VPS.
4. If you still have freezes it may be trait of low memory on remote VPS host, try to restart VPS or add memory :)
5. For iptables debuging use on client and server sides:
  * iptables -t raw -A OUTPUT -p udp -j TRACE
  * iptables -t raw -A PREROUTING -p udp -j TRACE
  * xtables-monitor --trace

# Spy-hunting

If you want to, you can identify additional ports used by spyware on your phone:

1. Connect to VPS by SSH
2. `sudo su`
3. `cat /var/log/syslog |grep p2p|grep -o "PROTO.*DPT=[0-9]*"|sort|uniq`

The above command will help you identify and list the unique ports that might be associated with spyware activity.
Ports 123, 137, 443, 5222, 80, and 8009 have been identified and closed for now.
Do not forget to identify the protocol as well.
