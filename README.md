# FreeBSD Jail/Bhyve Scripts

This repository contains scripts for managing [jails](https://docs.freebsd.org/en/books/handbook/jails/)
and [bhyve](https://docs.freebsd.org/en/books/handbook/virtualization/#virtualization-host-bhyve)
virtual machines on FreeBSD.

## Design Philosophy

There already exist many well-established tools for managing jails and VMs on FreeBSD,
such as [bastille](https://github.com/BastilleBSD/bastille) and [vm-bhyve](https://github.com/churchers/vm-bhyve).
You should probably use those, rather than my scripts.

I rolled my own for a few reasons:

- Doing things the "hard way" forces you to understand how the underlying technologies
  actually work.

- The existing tools use thousands of lines of code, with support for many features that
  I will never use.

- The FreeBSD community churns through jail managers as frequently as JavaScript developers
  churn through frontend frameworks.

There are two scripts: `jailctl` for managing FreeBSD jails, and `vmctl` for managing
bhyve virtual machines. Both scripts are written in FreeBSD's `/bin/sh` with no additional
dependencies.

The scripts treat jails and VMs in roughly the same fashion. Namely:

- Each jail/VM gets its own ZFS dataset, with two child datasets: `os` and `data`.

- The `os` dataset contains the operating system. The `data` dataset is used to
  store persisent application data. This allows you to to wipe and reprovision the
  OS disk easily.

- For jails, the `data` dataset is delegated to the jail and managed from within the
  jail itself.

- For VMs, both the `os` and `data` datasets are configured as `zvol` block devices.

- Each jail/VM gets a virtual NIC tagged with the specified VLAN ID. For jails,
  VNET is used with an `epair(5)` interface. For VMs, a `tap(4)` interface is
  automatically created.

- `vlan(4)` and `bridge(5)` interfaces are created as needed, using the physical
   interface specified by `$TRUNK_INTERFACE` (default is `lagg0`).

- For jails, static IPs and SSH keys configured by modifying files within the jail
  directly. For VMs, a `cloud-init` seed ISO is generated for guests that support it.

- If no IP configuration is given, DHCP is assumed.

## Installation

Simply copy the scripts to `/usr/local/sbin`.

    install -m 0555 jailctl /usr/local/sbin/jailctl
    install -m 0555 vmctl   /usr/local/sbin/vmctl

## Usage

TLDR for how to use both tools follows.

### jailctl

First, initialize the jail dataset:

    # jailctl init

A FreeBSD rootfs template was automatically created for you:

    # jailctl list-templates
    freebsd13

Create a new jail from the `freebsd13` template, using DHCP on VLAN 199 and our
SSH key:

    # jailctl create -v 199 -k ~/.ssh/id_ed25519.pub myjail1 freebsd13

Our `myjail1` jail should now be running:

    # jailctl list
    JAIL     STATUS
    myjail1  running

Open a shell within the jail:

    # jailctl shell myjail1
    root@myjail1:~ #

Run a command within the jail:

    # jailctl exec myjail1 ifconfig jail0 | grep inet
    inet 10.11.199.109 netmask 0xffffff00 broadcast 10.11.199.255

Let's create another jail, this time with a static IP and resource limits. We'll
use a ZFS quota of 32 GB, a memory limit of 4 GB, and a CPU limit of 400.

The CPU limit is given in terms of percentage per core. A limit of 400 means that
our jail can hog at most 4 CPU cores. See `man 8 rctl` for more information.

    # jailctl create                                   \
       -v 199                                          \
       -a 10.11.199.17 -n 255.255.255.0 -g 10.11.199.1 \
       -r 8.8.8.8 -r 8.8.4.4                           \
       -s sub.example.com -s example.com               \
       -c 400 -m 4G -q 32G                             \
       myjail2 freebsd13

Show jail configuration:

    # jailctl show myjail2
    ------------------------- JAIL CONFIGURATION -------------------------
    myjail2 {
      path = "/usr/local/jails/$name";
      host.hostname = "$name.test.example.com";

      exec.created  = "zfs set jailed=on zroot/jails/$name/data";
      exec.created += "zfs jail $name zroot/jails/$name/data";
      exec.start    = "zfs mount -a";
      exec.start   += "/bin/sh /etc/rc";
      exec.stop     = "/bin/sh /etc/rc.shutdown";
      exec.clean;

      exec.system_user = "root";
      exec.jail_user   = "root";

      mount.devfs;
      devfs_ruleset = "5";

      allow.mount     = true;
      allow.mount.zfs = true;
      enforce_statfs  = 1;

      vnet;
      vnet.interface = "ej_myjail2";
      exec.prestart  = "jailctl _create-epair $name vlan199 bridge199";
      exec.poststop  = "jailctl _destroy-epair $name";

      exec.poststop += "rctl -r jail:$name:";
      exec.prestart += "rctl -a jail:$name:pcpu:deny=400";
      exec.prestart += "rctl -a jail:$name:vmemoryuse:deny=4G";
    }

    ---------------------------- ZFS DATASET -----------------------------
    NAME                      QUOTA   USED  AVAIL  MOUNTPOINT
    zroot/jails/myjail2/os      24G   248K  24.0G  /usr/local/jails/myjail2
    zroot/jails/myjail2/data    32G    96K  32.0G  none

Show running jail status:

    # jailctl status myjail1
    ---------------------------- JAIL STATUS -----------------------------
    jid  name     path                      osrelease     host.hostname
    3    myjail1  /usr/local/jails/myjail1  13.2-RELEASE  myjail1.test.example.com

    ---------------------------- ZFS DATASET -----------------------------
    NAME  QUOTA  USED  AVAIL  MOUNTPOINT
    os    24G    260K  24.0G  /usr/local/jails/myjail1
    data  8G     96K   8.00G  none

    --------------------------- RESOURCE USAGE ---------------------------
    memoryuse  maxproc  openfiles  vmemoryuse  swapuse  pcpu  readbps  writebps  readiops  writeiops
    13M        5        40         64M         0        0     0        0         0         0

    ----------------------------- PROCESSES ------------------------------
    USER    PID %CPU %MEM   VSZ  RSS TT  STAT STARTED    TIME COMMAND
    root  25013  0.0  0.0 13152 2600  -  SsJ  09:21   0:00.00 dhclient: system.syslog (dhclient)
    root  25016  0.0  0.0 13152 2712  -  SsJ  09:21   0:00.00 dhclient: jail0 [priv] (dhclient)
    _dhcp 25095  0.0  0.0 13156 2756  -  SCsJ 09:21   0:00.00 dhclient: jail0 (dhclient)
    root  25218  0.0  0.0 12868 2716  -  SsJ  09:22   0:00.00 /usr/sbin/syslogd -ss
    root  25257  0.0  0.0 12908 2496  -  SsJ  09:22   0:00.00 /usr/sbin/cron -s

Take a ZFS snapshot:

    # jailctl snapshot myjail1

List available snapshots:

    # jailctl list-snapshots myjail1
    DATASET  SNAPSHOT             USED  REFER
    data     2023-10-06T07:23:22  0B    96K
    os       2023-10-06T07:23:22  0B    503M

Rollback to a previous snapshot:

    # jailctl rollback myjail1 2023-10-06T07:23:22

Delete a snapshot:

    # jailctl destroy-snapshot -y myjail1 2023-10-06T07:23:22

Reprovision a jail's OS dataset from a template:

    # jailctl reprovision -y myjail1 freebsd13

Delete a jail:

    # jailctl destroy -y myjail2

### vmctl

First, initialize the bhyve dataset and install dependencies:

    # vmctl init

Download an ISO:

    # vmctl download-iso https://download.freebsd.org/releases/amd64/amd64/ISO-IMAGES/13.2/FreeBSD-13.2-RELEASE-amd64-bootonly.iso

List available ISOs:

    # vmctl list-isos
    FreeBSD-13.2-RELEASE-amd64-bootonly
    debian-12.1.0-amd64-netinst

Create a new VM with VLAN ID 199, 2 CPUs, 4 GB of RAM, and 16 GB of disk space, booting
from an ISO file:

    # vmctl create                             \
        -v 199                                 \
        -c 2 -m 4G -q 16G                      \
        -i FreeBSD-13.2-RELEASE-amd64-bootonly \
        myvm1

Connect to a VM's serial console (disconnect using `~.`):

    # vmctl console myvm1

Download a disk image:

    # vmctl download-template https://cloud.debian.org/images/cloud/bookworm/latest/debian-12-generic-amd64.raw

List available template images:

    # vmctl list-templates
    debian-12-generic-amd64
    freebsd-13.2-ufs-2023-04-22

Create a VM from a template image, using [cloud-init](https://cloudinit.readthedocs.io/en/latest/) to
configure SSH keys and static IP:

    # vmctl create                           \
        -v 199                               \
        -a 10.11.199.64 -g 10.11.199.1 -p 24 \
        -r 8.8.8.8 -r 4.4.4.4                \
        -s sub.example.com -s example.com    \
        -k ~/id_ed25519.pub                  \
        -c 2 -m 2G -q 32G                    \
        myvm2                                \
        debian-12-generic-amd64

List running VMs:

    # vmctl list
    VM     STATUS
    myvm1  running
    myvm2  running

Show VM configuration:

    # vmctl show myvm1
    ------------------------- BHYVE CONFIGURATION -------------------------
    name=myvm1
    uuid=0c7ea844-645f-11ee-baf1-b496910daabc
    cpus=2
    memory.size=4G
    acpi_tables=true
    destroy_on_poweroff=true
    keyboard.layout=us_unix
    rtc.use_localtime=false
    x86.strictmsr=true
    x86.vmexit_on_hlt=true
    x86.vmexit_on_pause=true
    pci.0.0.0.device=hostbridge
    pci.0.1.0.device=lpc
    pci.0.2.0.device=virtio-net
    pci.0.2.0.backend=tap_myvm1
    pci.0.2.0.mac=1e:f0:9a:0d:aa:bc
    pci.0.3.0.device=nvme
    pci.0.3.0.path=/dev/zvol/zroot/bhyve/%(name)/os
    pci.0.4.0.device=nvme
    pci.0.4.0.path=/dev/zvol/zroot/bhyve/%(name)/data
    pci.0.5.0.device=ahci
    pci.0.5.0.port.0.type=cd
    pci.0.5.0.port.0.path=/usr/local/bhyve/%(name)/seed.iso
    pci.0.5.0.port.0.ro=true
    lpc.com1.path=/dev/nmdm_%(name)A
    lpc.bootrom=/usr/local/share/uefi-firmware/BHYVE_UEFI.fd
    lpc.bootvars=/usr/local/bhyve/myvm1/uefi-vars.fd
    vmctl.vlan=199

    ---------------------------- ZFS DATASET -----------------------------
    NAME                    VOLSIZE   USED     REFER
    zroot/bhyve/myvm1/os        24G  24.1G       56K
    zroot/bhyve/myvm1/data      16G  16.1G       56K

Show running VM status:

    # vmctl status myvm1
    ----------------------------- BHYVE PROCESS ------------------------------
    USER   PID %CPU %MEM     VSZ   RSS TT  STAT STARTED    TIME COMMAND
    root 27652  0.0  0.2 4258836 51716  1- IC   11:43   0:40.09 bhyve: myvm1 (bhyve)

    ---------------------------- ZFS DATASET -----------------------------
    NAME  VOLSIZE  USED   REFER
    os    24G      24.1G  56K
    data  16G      16.1G  56K

    --------------------------- RESOURCE USAGE ---------------------------
    memoryuse  vmemoryuse  pcpu  readbps  writebps  readiops  writeiops
    50M        4159M       0     0        0         0         0

Take a ZFS snapshot:

    # vmctl snapshot myvm1

List available snapshots:

    # vmctl list-snapshots myvm1
    DATASET  SNAPSHOT             USED  REFER
    data     2023-10-06T07:55:21  0B    56K
    os       2023-10-06T07:55:21  0B    56K

Rollback to a previous snapshot:

    # vmctl rollback myvm1 2023-10-06T07:55:21

Delete a snapshot:

    # vmctl destroy-snapshot -y myvm1 2023-10-06T07:55:21

Reprovision a VM's OS zvol from a template:

    # vmctl reprovision -y myvm1 debian-12-generic-amd64

Delete a VM:

    # vmctl destroy -y mvvm1
