# pi-hole2vpn

## Why

1. I found only pre-built droplets on DigitalOcean and a lot of instructions about setting up Pi-hole and WireGuard, but I do not want to configure
    everything each time with so many settings. Ansible is an easy method to write some "setup notes" one time, and DigitalOcean has traffic limits
    on droplets.
2. When I use ad-blocker DNS, my phone stays cool and works for a full day :) It is truly the best method to increase battery lifetime.
3. I dislike spyware.

Look at screenshot. 80% requests it is spyware!
<p align="center">
  <img src="https://github.com/3DRaven/pi-hole2vpn/blob/master/images/Pi-hole-on-phone.png" width="150" height="280">
</p>

## How to use

Prerequirements:

1. Remote host with ssh access
2. Ubintu 22.04 on remote host (tested only with Ubuntu and multiple VPS providers)

Install steps:

1. On LOCAL computer 
```bash
sudo apt-get install ansible
```
2. Edit `group_vars/vpn`. It is file with main settings.
3. Edit `inventory` file to add IP of your remote hosts to install VPN+Pihole, in this file possible to set ssh access params
4. Execute command on LOCAL computer (in dir with deploy.yml file) 
```bash
ansible-playbook --ask-become-pass ./deploy.yml
```
If you do not need some actions just use tags. Available tags: [user_cration,vpn_installation,docker_installation,pi_hole_installation,adblock_add,adblock_remove,disable_ubuntu_user]
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
4. Zram is installed. It is a good method to expand VPS RAM on the remote host.
5. WireGuard is installed on the remote host.
6. Client configuration files are generated on the localhost.
7. If default `group_vars/vpn->wireguard_listen_port` port is blocked all traffic from ports `group_vars/vpn->fallback_wireguard_listen_ports` will be redirected to `group_vars/vpn->wireguard_listen_port`

## Using VPN from phone:

1. Install wireguard client to phone
2. Scan QR code of any client from client dir (config_*.txt files it is QR codes) and connect to VPN
3. Open http://pi.hole/admin in browser (access only from VPN, password from `group_vars/vpn->pi_hole_admin_password`)

## Using from local computer

### Ubuntu

Command ro install wireguard
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

1. If you have problems with freezes tasks try to comment `inventory->ssh_connection` options. It is slower but may resolve some problems.
2. Do not forget to open ports 22 (SSH), 51820 (default VPN) on providers firewall
3. If you still have freezes it may be trait of low memory on remote VPS host, try to restart VPS or add memory :)