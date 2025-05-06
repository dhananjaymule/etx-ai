# SOP: Kdump Configuration for PCS Cluster (Delayed Fencing), Early Kdump, and Network Targets

## 1. Purpose

This document provides the Standard Operating Procedure (SOP) for configuring kdump on nodes in a PCS-managed cluster. It covers specific setups including:

- Handling kdump with delayed fencing

- Enabling early kdump for better capture in crash scenarios

- Configuring network-based crash dumps (NFS, SSH)

## 2. Scope

Applies to all Linux servers (RHEL 7/8/9) configured under PCS cluster management where crash recovery and consistent fencing behavior are critical.

## 3. Prerequisites

- Root or sudo access on all cluster nodes.
- PCS Cluster is properly configured and running.
- Access to a network share (NFS server or SSH server) for storing dumps.
- Packages installed: kexec-tools, fence-agents, nfs-utils (for NFS).

## 4. Configuration

### 4.1 Enable and Configure Kdump

Install Required Packages:

sudo yum install kexec-tools nfs-utils

Enable and Start kdump Service:

sudo systemctl enable kdump
sudo systemctl start kdump

Reserve Crashkernel Memory:

GRUB\_CMDLINE\_LINUX="crashkernel=512M ..."

sudo grub2-mkconfig -o /boot/grub2/grub.cfg
sudo reboot

### 4.2 Configure Early Kdump

Edit /etc/sysconfig/kdump:

KDUMP\_COMMANDLINE\_APPEND="earlykdump"

Optional minimal modules:

KDUMP\_KERNELVER=""
KDUMP\_KERNEL\_OPTIONS=""
KDUMP\_COMMANDLINE="irqpoll maxcpus=1 reset\_devices"

sudo kdumpctl rebuild

### 4.3 Configure Network Targets

Edit /etc/kdump.conf for NFS:

path /var/crash
core\_collector makedumpfile -l --message-level 1 -d 31
nfs &lt;nfs-server&gt;:/exported/path

Example:

nfs 10.10.10.5:/kdump

For SSH:

ssh user@remote-server

ssh-keygen
ssh-copy-id user@remote-server

sudo systemctl restart kdump

### 4.4 Configure PCS Cluster Delayed Fencing for Crashdump

#### 4.4.1 Setting fencing delay

pcs stonith update &lt;stonith-device-name&gt; pcmk\_delay\_max=60

#### 4.4.2 Using kdump\_helper for Crash Notification

fence\_kdump\_nodes

sudo yum install fence-agents-kdump

pcs stonith create fence\_kdump fence\_kdump\_nodes pcmk\_host\_list="node1 node2" pcmk\_monitor\_action="metadata" op monitor interval=30s

## 5. Example Files

### /etc/kdump.conf

path /var/crash
core\_collector makedumpfile -l --message-level 1 -d 31
default shell
nfs 10.10.10.5:/kdump
fence\_kdump\_nodes

### /etc/sysconfig/kdump

KDUMP\_KERNELVER=""
KDUMP\_COMMANDLINE="irqpoll maxcpus=1 reset\_devices"
KDUMP\_COMMANDLINE\_APPEND="earlykdump"
KDUMP\_IMG\_REBUILD="yes"

### PCS Fencing Device (Example)

pcs stonith create my-fence fence\_ipmilan ipaddr="10.0.0.5" login="admin" passwd="password" pcmk\_delay\_max=60 op monitor interval=60s
pcs stonith create fence\_kdump fence\_kdump\_nodes pcmk\_host\_list="node1 node2"

## 6. Validation Steps

Check kdump service:

sudo systemctl status kdump

Simulate crash (test carefully on test node):

echo c &gt; /proc/sysrq-trigger

Check if dump is generated to remote target.
Confirm cluster does not fence immediately (delayed fencing works).

## 7. Troubleshooting

- Ensure enough crashkernel memory is reserved.
- Verify NFS/SSH connection to dump target.
- Monitor fencing logs: /var/log/pacemaker.log
- Test fence\_kdump separately with:

fence\_kdump\_send -p 7410 -i eth0 -m reboot

End of Document.

### 4.5 Handling System Hung and NMI Interrupts

In certain cases where the system hangs and no crash is triggered, it is necessary to capture crash dumps manually using an NMI (Non-Maskable Interrupt).

Enable sysrq and NMI support:

1. Edit /etc/sysctl.conf to enable sysrq:

kernel.sysrq = 1

Apply the change immediately:

sudo sysctl -p

2. Configure NMI to trigger a crash:

Add the following to your GRUB configuration (/etc/default/grub):

GRUB\_CMDLINE\_LINUX="... nmi\_watchdog=1 crash\_kexec\_post\_notifiers"

Update grub and reboot the system:

sudo grub2-mkconfig -o /boot/grub2/grub.cfg
sudo reboot

Testing NMI Crash:

Use IPMI to trigger an NMI remotely or press the NMI button if available. For IPMI:

ipmitool chassis power diag

After issuing the NMI, the system should trigger kdump and generate a crash dump.