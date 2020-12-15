# Alpine Linux Reverse Proxy Setup

**Business Goal:**
The goal of this project is to outline the setup of the reverse proxy used for a blue/green setup of Kubernetes clusters. [Alpine Linux Extended](https://alpinelinux.org/downloads/) was selected as it comes with preinstalled packages that are suitable for routers and servers. Alpine was chosen as it is lightweight, fast, and doesn't contain a lot of bloat.

> This is just an example. To use add certs (bundle.pem/server.key) to SSL. Update the nginx configs to include real IP Address and Ports to Apps running in K8s (assuming nodePort service was used in K8s)

## Project Requirements

- Small harddrive space footprint. Less than 4GB & 4 GB of RAM
- SSH Server
- Basic Tools for troubleshooting: cat, curl, htop, hping3, iperf, nano, netcat, nmap, nslookup, ping, tar, vi, and wget
- Hyper-v guest services
- A tool to stress test the proxy and the max number of connections per second (see the Siege section at the bottom of this readme).

--- 

## Important Notes on Infrastructure

This was originally designed to allow for switching from one kubernetes cluster to another. This would completely cut off traffic to the old cluster. This is built for on prem k8s clusters. For a hybrid cloud or full cloud solution using AWS look into an AWs Load balancer. 

In it's current state it can round robin traffic to the one cluster once. If it's desired to switch clusters, comment the files out in nginx.conf and run `sudo nginx -t` to test the config change & run `sudo nginx -s reload` to apply the config change.

> This was successfully used on a cluster that used a [nodePort k8s service strategy](https://kubernetes.io/docs/concepts/services-networking/service/). 

---

## Setup Hyper V

- Download Apline Linux Standard -alpine-extended-3.8.0-x86.iso
- Setup new hyper-v with the ISO

  - Generation 1 Hyper VM
  - 4096 MB Memory
  - Setup to external network
  - 4 GB HDD

---

## Install Alpine Linux

Follow instructions below or [online](https://wiki.alpinelinux.org/wiki/Install_to_disk).

> _The instructions below are for a Windows 10 local VM. Some settings may need to change in order to run on certain servers. For example, ip address may be a static address on the the server and would need configured in the setup process_.

```
login: root

setup-alpine
keyboard-layout: us
available variants: us
hostname: press enter
Network interface card: press enter for eth0
ip address for eth0: press enter dhcp
Manual Network Config: press enter for no

Set password: alpine (change these)
Reenter password: alpine (change these)

Choose time zone: US/ then sub zone of Central

HTTP/FTP proxy URL: press enter for none

Choose a mirror: F for fastest

Which SSH Server: enter for openssh
Which NTP: enter for chrony

Which disks would you like to use: sda
How would you like to use it: sys
Erase the above disk and continue: y

Reboot
```

After the reboot is complete. **Shutdown the VM and eject the ISO.** Then power up the VM. Login.

---

## Setup Admin, Add APKs and Set Sudo Access

Run `adduser admin` set the password. **This step is vital for SSH Access. Root user cannot use SSH**

> change a users password by using root and `passwd admin`

---

### Setup VM APK Packages

**Edit the list of repositories**
Enter `cd /etc/apk/` then use vi to edit the repository list. `vi repositories`

Hit insert on the keyboard and copy paste the following:

```shell
http://mirrors.gigenet.com/alpinelinux/v3.8/main
http://dl-cdn.alpinelinux.org/alpine/latest-stable/community
```

Hit escape on the keyboard and then enter: `:wq` to exit.

---

**Adding/Updating Packages**

run `apk update && apk upgrade`

type `su` and enter in the root password. then run:
`apk add ca-certificates curl e2fsprogs git htop screen tar tree tmux wget vim nano nginx sudo hvtools`

---

### Add admin to sudo user list

Add admin to the list of sudo users. Run `visudo`. Go to the User Privilege Specification Comment block and add `admin ALL=(ALL) ALL`. This is done by pressing the insert keyboard key, hitting escape when done typing, and then running `:wq`

> **It is important to add sudo users via visudo.** Other methods will break things.

---

## Add the following services

```shell
rc-update add hv_fcopy_daemon default
rc-update add hv_kvp_daemon default
rc-update add hv_vss_daemon default
rc-update add nginx default
```

Then run `reboot`

---

## Copy Index.html for Health Checking 

Create the nginx-status directory and copy the index.html file from the nginx-status directory in this repo

```shell
cd /var/www
mkdir nginx-status
sudo nano index.html
```

---

## Setup SSL/TLS

Create the ssl directory and copy the wild card cert from this repo into the directory

```shell
cd /home/admin
mkdir ssl
```

---

## Setup Nginx

The Nginx configurations are broken out into multiple files that are included into the main Nginx.conf. This produces smaller files making trouble shooting easier. Switching from blue to green is done by commenting out and uncommenting the other in the main nginx.conf located at `/etc/nginx/nginx.conf`

**Make Blue/Green Directories**
```shell
cd /home/admin/
mkdir blue-config
mkdir green-config
```

**Add files from this repo into the blue-config & green config directories.** The easiest way to do this is via `sudo nano blue-cofigName.conf`. The config files are located in this repository under [nginx-configs](./nginx-configs/). Alternatively, upload via FTP.


**Copy main Nginx from this repo: `./nginx-configs/nginx.conf`**

```shell
cd /etc/nginx
sudo nano nginx.conf
```

Paste in the nginx.conf file contents from this repo. Save, test the config `sudo nginx -t` & run `sudo nginx -s reload`. (_Delete the template with comments if needed_)

---

## Additional NGINX Commands

**Check for NGINX.conf for errors:**
`sudo nginx -t`

**How to quickly reload NGINX without downtime:**
`sudo nginx -s reload`

**How to stop the service:**
`rc-service nginx stop`

**How to start the service:**
`rc-service nginx start`

**How to fully reload the service:**
`rc-service nginx reload`

---

#### Useful Commands for trouble shooting NGINX

```shell
sudo grep -rnwi "listen" /etc/nginx
sudo fuser 80/tcp
sudo fuser -k 80/tcp KILLS EVERYTHING ON PORT 80
ps aux

sudo netstat -ltnp

ls /proc/  
ls /proc/PIDnumber
```

---

## Stress Testing with Docker SIEGE

We are using [Siege](https://www.joedog.org/siege-home/) to stress test our nginx proxy. We can do this using the docker-siege folder within this repository. Navigate to that folder and run `docker build -t docker-siege .` This will build the image. We can then use the docker container to stress test the connection with this command. `docker run --rm -t docker-siege -d1 -r10 -c25 https://10.192.60.75`. The siege link above describes the flags available.

- `-d1` sets a delay of 1 second between rounds. This ensures traffic comes in waves, not all at once.
- `-r10` sets the number of rounds. This case being 10 rounds.
- `-c25` sets the number of concurrent users. This case being 25.

This will output text similar to below

```shell
** SIEGE 3.0.5
** Preparing 25 concurrent users for battle.
The server is now under siege.. done.

Transactions: 250 hits
Availability: 100.00 %
Elapsed time: 8.07 secs
Data transferred: 0.07 MB
Response time: 0.01 secs
Transaction rate: 30.98 trans/sec
Throughput: 0.01 MB/sec
Concurrency: 0.27
Successful transactions: 250
Failed transactions: 0
Longest transaction: 0.02
Shortest transaction: 0.00
```

> This is a copy of [Docker Siege](https://hub.docker.com/r/yokogawa/siege/) on Dockerhub. It can be alternatively ran via `docker run --rm -t yokogawa/siege -d1 -r10 -c25 https://10.192.60.75`.

---

## Additional Resources

- [Search for Alpine Packages](https://pkgs.alpinelinux.org/packages)
- [Alping Linux Setup](https://wiki.alpinelinux.org/wiki/Nginx)
