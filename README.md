# kube-lxc

Deployment instructions for creating a kubernetes cluster where one or more nodes are LXC containers in Proxmox

## Architecture

- 2 masters and 1 worker
- Both masters are Rocky Linux LXC containers on my Proxmox machine ([fitlet2](https://fit-iot.com/web/products/fitlet2/) with Intel Atom E3950)
- First master:
  - IP: 10.1.20.11
  - Hostname: k8m01
- Second master:
  - IP: 10.1.20.12
  - Hostname: k8m02
- The worker is an [HP S01-pf1013w](https://forums.serverbuilds.net/t/official-hp-s01-pf1013w-owners-thread-and-review/9070) running Rocky Linux
  - IP: 10.1.20.21
  - Hostname: k8w01

## Prepare Bastion Host

Presumably, you're going to run your `kubectl` commands from a machine not in the cluster. That's your bastion host. In my case, it's an openSUSE Tumbleweed machine using the Alacritty terminal.

### Install your terminfo (optional)

Since I'll be doing SSH things on my Proxmox server and on all Kubernetes nodes, I'll need to install the Alacritty terminfo on all machines.

We'll start with running this on my Proxmox server:

`apt-get update && apt-get install git`

`git clone https://github.com/alacritty/alacritty.git`

`cd alacritty`

`tic -xe alacritty,alacritty-direct extra/alacritty.info`

### Prepare SSH Keys and Config

Before doing anything, you'll want to prepare your SSH keys

I didn't want to use the default key names of `id_rsa` and `id_rsa.pub` so I ran the following command:

`ssh-keygen -t rsa -b 4096 -N '' -f ~/.ssh/k8s_rsa`

Since every one of my kubernetes nodes will have a hostname that starts with `k8`, I'm making the SSH config apply to only the `k8*` hosts.

Notice that I use `cat` to append because I don't need root privileges.

```sh
cat >> ~/.ssh/config<< EOF
Host k8*
    UserKnownHostsFile /dev/null
    StrictHostKeyChecking no
    IdentitiesOnly yes
    ConnectTimeout 0
    ServerAliveInterval 30
    IdentityFile ~/.ssh/k8s_rsa
EOF
```

Finally, copy the public key to your Proxmox host (10.1.20.10 in my case) so you can use it in the next session.

`scp ~/.ssh/k8s_rsa.pub root@10.1.20.10:/root/.ssh/k8s_rsa.pub`

You didn't actually set up this key to be used for SSH authentication to the Proxmox host itself. You are simply making the public key available on the host so that it may be used in containers.

If you want to also use this key for authentication into your Proxmox host, use the following command (in addition to the command above):

`ssh-copy-id -i ~/.ssh/k8s_rsa.pub root@10.1.20.10`

### Prepare hosts file

My four hosts:

Host Purpose  |     IP     |       FQDN      | hostname
--------------|------------|-----------------|---------
Bastion       | 10.1.20.40 | lizard.paw.blue | lizard
K8s Master 01 | 10.1.20.11 | k8m01.paw.blue  | k8m01
K8s Master 02 | 10.1.20.12 | k8m02.paw.blue  | k8m02
K8s Worker 01 | 10.1.20.21 | k8w01.paw.blue  | k8w01

So I want to update the /etc/hosts file on my bastion machine like this:

```sh
sudo tee -a /etc/hosts > /dev/null <<EOT
10.1.20.40 lizard.paw.blue lizard
10.1.20.11 k8m01.paw.blue k8m01
10.1.20.12 k8m02.paw.blue k8m02
10.1.20.21 k8w01.paw.blue k8w01

# K8s Endpoints
10.1.20.11 k8ep.paw.blue k8ep
10.1.20.12 k8ep.paw.blue k8ep
EOT
```

Notice that I use `tee` to append because I need root privileges.

### Disable swap

This is directly from [this page](https://blog.lbdg.me/proxmox-best-performance-disable-swappiness/)

```sh
# Check the current value
cat /proc/sys/vm/swappiness

# Define the new one
sysctl vm.swappiness=0

# Disable SWAP, it'll take some times to clean the SWAP area
swapoff -a

# Enable the SWAP with the new value
swapon -a

# Check if well applied
cat /proc/sys/vm/swappiness
```

I found out later that the above wasn't enough. **I also needed to comment out the swap line in `/etc/fstab`**

## Master Node Initial Setup (LXC)

Sources:

- [Prepare Proxmox host](https://du.nkel.dev/blog/2021-03-25_proxmox_docker/)
- [Provision LXC container]([this](https://gist.github.com/tinoji/7e066d61a84d98374b08d2414d9524f2))
- [More LXC provisioning](https://medium.com/geekculture/a-step-by-step-demo-on-kubernetes-cluster-creation-f183823c0411)
- [Install cluster](https://computingforgeeks.com/install-kubernetes-cluster-on-rocky-linux-with-kubeadm-crio/)

> On Proxmox, the overlay and aufs Kernel modules must be enabled to support Docker-LXC-Nesting.

`echo -e "overlay\naufs" >> /etc/modules-load.d/modules.conf`

> Reboot Proxmox and verify that the modules are active:

`lsmod | grep -E 'overlay|aufs'`

Also in Proxmox, prepare root password variable for master node

`echo -n Password: && read -s password && echo`

The above command will prompt you for a password. Enter it and hit return.

### Semi Automatic deployment

You can automatically deploy the two nodes with separate commands as shown below, or you can skip to the next section where it's done fully automatically.

In Proxmox, use `pct` command to create the master node.

```sh
pct create 102 /var/lib/vz/template/cache/rockylinux-8-default_20210929_amd64.tar.xz \
    --arch amd64 \
    --ostype centos \
    --hostname k8m01 \
    --cores 4 \
    --memory 4096 \
    --swap 0 \
    --storage local-lvm \
    --password $password \
    --ssh-public-keys /root/.ssh/k8s_rsa.pub \
    --net0 name=eth0,bridge=vmbr0,firewall=0,gw=10.1.20.1,ip=10.1.20.11/24,type=veth \
    --unprivileged 1 \
    --onboot 1 \
    --features nesting=1,keyctl=1
```

Now use the following command to start the new container, wait for 10 seconds, increase the default size of 4G to 12G by adding 8G, enter the container, and bootstrap it.

```sh
pct start 102 &&\
sleep 10 &&\
pct resize 102 rootfs +8G &&\
pct exec 102 -- bash -c\
    "dnf -y update &&\
    dnf -y install vim git wget epel-release openssh-server &&\
    systemctl start sshd &&\
    systemctl enable sshd"
```

Optional step if you use Alacritty on your bastion host:

```sh
pct exec 102 -- bash -c\
    "git clone https://github.com/alacritty/alacritty.git &&\
    cd alacritty &&\
    tic -xe alacritty,alacritty-direct extra/alacritty.info"
```

You would then need to run these commands again for the second node.

### Fully Automatic Deployment

Alternatively, you can create both nodes with a single bash script.

This will create two nodes as follows:

- id 102, k8m01, 10.1.20.11
- id 103, k8m02, 10.1.20.12

```sh
for ((n=102,host=1;n<=103;n++,host++))
do
  pct create $n /var/lib/vz/template/cache/rockylinux-8-default_20210929_amd64.tar.xz \
    --arch amd64 \
    --ostype centos \
    --hostname k8m0$host \
    --cores 4 \
    --memory 4096 \
    --swap 0 \
    --storage local-lvm \
    --password $password \
    --ssh-public-keys /root/.ssh/k8s_rsa.pub \
    --net0 name=eth0,bridge=vmbr0,firewall=0,gw=10.1.20.1,ip=10.1.20.1$host/24,type=veth \
    --unprivileged 1 \
    --onboot 1 \
    --features nesting=1,keyctl=1 &&\
    pct start $n &&\
    sleep 10 &&\
    pct resize $n rootfs +8G &&\
    pct exec $n -- bash -c\
        "sudo dnf -y update &&\
        dnf -y install vim git wget epel-release openssh-server &&\
        systemctl start sshd &&\
        systemctl enable sshd &&\
        git clone https://github.com/alacritty/alacritty.git &&\
        cd alacritty &&\
        tic -xe alacritty,alacritty-direct extra/alacritty.info"
done
```

## Worker Node Initial Setup (Physical)

This is trickier because you will need to manually enter these commands until you have network access.

### Network Setup

In order to get the ethernet adapter's name, enter `ip a` on the host.

You need to edit the config for that adapter so that it starts on boot.

My network adapter is called `enp2s0` so I used the `sed` command to change `ONBOOT` from 'no' to 'yes' in `/etc/sysconfig/network-scripts/ifcfg-enp2s0`

`sed -i 's/ONBOOT=no/ONBOOT=yes/g' /etc/sysconfig/network-scripts/ifcfg-enp2s0`

I also had to do this so it would always start the NIC.

`nmcli connection modify enp2s0 ipv4.method auto`

`nmcli connection down enp2s0; sudo nmcli connection up enp2s0`

### Get SSH Access

`dnf -y update`

`dnf -y install openssh-server`

`systemctl start sshd`

`systemctl enable sshd`

### Set Hostname

`hostnamectl set-hostname k8w01`

### Install Basic Utilities

At this point, you should be able to SSH into the host and run this command

`dnf -y install vim git wget epel-release`

### Alacritty terminfo (optional)

If you use Alacritty on your bastion host, run this on your worker node:

```sh
git clone https://github.com/alacritty/alacritty.git &&\
    cd alacritty &&\
    tic -xe alacritty,alacritty-direct extra/alacritty.info
```

## Bastion Host K8s Setup

I used [this](https://computingforgeeks.com/install-kubernetes-cluster-on-rocky-linux-with-kubeadm-crio/) guide.

I won't reiterate all the steps from that guide, but here are some of them, as there were some occasional deviations. For some of the steps, I included instructions for both openSUSE and Rocky Linux bastion hosts.

### Install Python 3.9

Rocky Linux only: `sudo dnf -y install python39`

openSUSE only: `sudo zypper in python39`

Both OSes:

`sudo pip3 install setuptools-rust wheel`

`sudo pip3 install --upgrade pip`

### Install Ansible

`sudo python3.9 -m pip install ansible`

`ansible --version`

You should see something like this in Rocky Linux:

```sh
ansible [core 2.12.1]
  config file = None
  configured module search path = ['/home/nova/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/local/lib/python3.9/site-packages/ansible
  ansible collection location = /home/nova/.ansible/collections:/usr/share/ansible/collections
  executable location = /usr/local/bin/ansible
  python version = 3.9.6 (default, Nov  9 2021, 13:31:27) [GCC 8.5.0 20210514 (Red Hat 8.5.0-3)]
  jinja version = 3.0.3
  libyaml = True
```

You should see something like this in openSUSE:

```sh
ansible [core 2.12.1]
  config file = None
  configured module search path = ['/home/will/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/local/lib/python3.9/site-packages/ansible
  ansible collection location = /home/will/.ansible/collections:/usr/share/ansible/collections
  executable location = /usr/local/bin/ansible
  python version = 3.9.9 (main, Nov 17 2021, 09:50:59) [GCC]
  jinja version = 3.0.3
  libyaml = True
```

### SSH Auth on Worker

If you used the `pct create` commands above, then the master nodes already authenticate with your `k8s_rsa` key. The following command will do the same for the worker node.

`ssh-copy-id -i ~/.ssh/k8s_rsa.pub root@k8w01`

### Hostname Check

Check hostname value on each host from Bastion

`ssh root@k8m01 'hostnamectl'`

`ssh root@k8m02 'hostnamectl'`

`ssh root@k8w01 'hostnamectl'`

## Ansible Setup

For my use case, I relied heavily upon [this](https://computingforgeeks.com/install-kubernetes-cluster-on-rocky-linux-with-kubeadm-crio/) guide again, this time modifying the playbooks for my own use.

I cloned the repo and then modified the files for my own Kubernetes architecture.

I won't repeat all of the instructions, but you can clone the repo and modify for your own needs.

Some things I ran while trying to use the playbook as-is:

### Swap problems

It hung on disabling swap on my LXC containers. I made sure they have swapoff and then commented out the "disable swap" task.

### SELinux problems

SELinux is disabled on my Proxmox host, and I don't want to enable it, so I replaced this:

```yaml
- name: Put SELinux in permissive mode
  ansible.posix.selinux:
    policy: targeted
    state: "{{ selinux_state }}"
```

With this:

```yaml
- name: Check uname 
  shell: "uname -r"
  register: uname

- name: Put SELinux in permissive mode if enforcing
  command: sed -i 's/SELINUX=enforcing/SELINUX=permissive/g' /etc/selinux/config
  register: selinux_config
  when: '"pve" not in uname.stdout'
```

### Kernel module problems

I ran into an issue with the `load_kernel_modules_sysctl.yml` tasks on my LXC hosts. Apparently, I needed to install the modules on the Proxmox (PVE) host first.

I ran the following to install them on my Proxmox host:

```sh
for value in br_netfilter overlay ip_vs ip_vs_rr ip_vs_wrr ip_vs_sh nf_conntrack
do
    modprobe $value
    lsmod | grep $value
done
```

Explanation of script:

- `modprobe $value` will load the module

- `ls mod | grep $value` will find the active module

In the output of my script, I didn't see br_netfilter, but according to [this post](https://forum.proxmox.com/threads/docker-support-in-proxmox.27474/post-295237) in the Proxmox forums, "br_netfilter no need to load on pve kernels."

In that post, he includes this command and output:

> $ grep 'BRIDGE_NETFILTER' /boot/config-$(uname -r)

> CONFIG_BRIDGE_NETFILTER=y

There's more discussion on modules [here](https://gist.github.com/triangletodd/02f595cd4c0dc9aac5f7763ca2264185).

Now that you've loaded the modules on the Proxmox host, you do not need to load them on the container. However, because the ansible playbook wants to verify that `br_netfilter` is loaded, I've separated out the module loading into two separate tasks and made sure the LXC containers won't try to load `br_netfilter` (because they have `pve` in their output of `uname -r`).

Here's what it looked like before:

```yaml
- name: Load required modules
  community.general.modprobe:
    name: "{{ item }}"
    state: present
  with_items:
    - br_netfilter
    - overlay
    - ip_vs
    - ip_vs_rr
    - ip_vs_wrr
    - ip_vs_sh
    - nf_conntrack
```

And this is after:

```yaml
- name: Load required modules (all hosts)
  community.general.modprobe:
    name: "{{ item }}"
    state: present
  with_items:
    - overlay
    - ip_vs
    - ip_vs_rr
    - ip_vs_wrr
    - ip_vs_sh
    - nf_conntrack

- name: Load required modules (non-PVE hosts)
  community.general.modprobe:
    name: "{{ item }}"
    state: present
  with_items:
    - br_netfilter
  when: '"pve" not in uname.stdout'
```

### Getting hosts right

My `./roles/kubernetes-bootstrap/templates/hosts.j2` looks like this:

```sh
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

{% for host in groups['k8snodes'] %}
{{ hostvars[host]['ansible_facts']['default_ipv4']['address'] }} {{ hostvars[host]['ansible_facts']['fqdn'] }} {{ hostvars[host]['ansible_facts']['hostname'] }}
{% endfor %}

# K8s Endpoints
10.1.20.11 k8ep.paw.blue k8ep
10.1.20.12 k8ep.paw.blue k8ep
```

So it will get all three hosts from the `hosts` file as well as two additional endpoint FQDNs.

## Finishing the playbook

Finally, after customizing the Ansible playbook for my environment, I ran the following to execute it:

`ansible-playbook -i hosts k8s-prep.yml`

Note: You can run this playbook multiple times. I had to do it many, many times while troubleshooting.

## Bootstrap Kubernetes Control Plane

I won't rewrite the original author's steps, but I will note what commands I ran below:

Prevent module load failure message:

```sh
for ((node=102;node<=103;node++))
do
  pct push $node /boot/config-$(uname -r) /boot/config-$(uname -r)
done
```

Doing a dry run on one node:
`ssh root@k8m01 kubeadm init --dry-run --apiserver-advertise-address=10.1.20.11 --apiserver-cert-extra-sans=10.1.20.11 --control-plane-endpoint=k8ep.paw.blue --pod-network-cidr=192.168.0.0/16 --kubernetes-version=1.23.1 > dryrun.txt`


Read over the file, and if it looks good, proceed to a real run on both nodes:

```sh
for ((node=1;node<=2;node++))
do
  ssh root@k8m0$node kubeadm init --apiserver-advertise-address=10.1.20.1$node --apiserver-cert-extra-sans=10.1.20.1$node --control-plane-endpoint=k8ep.paw.blue --pod-network-cidr=192.168.0.0/16 --kubernetes-version=1.23.1
done
```

If it does not complete successfully, you can do this to undo:

```sh
for ((node=1;node<=2;node++))
do
  ssh root@k8m0$node kubeadm reset
done
```

If it completes successfully, it will tell you to run these commands on each master **if you're not using root**:

```sh
mkdir -p $HOME/.kube &&\
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config &&\
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

I'm running as root user, so I ran this:

```sh
for ((node=1;node<=2;node++))
do
  ssh root@k8m0$node export KUBECONFIG=/etc/kubernetes/admin.conf
done
```

## License

Distributed under the MIT License. See LICENSE for more information.

## Credit

Thanks to:

- [This guide](https://computingforgeeks.com/install-kubernetes-cluster-on-rocky-linux-with-kubeadm-crio/) by Josphat Mutai
