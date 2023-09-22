# freebsd-scripts
My system administration utilites for FreeBSD.

## jailctl

`jailctl` is a thin wrapper around various `jail(8)`, `zfs(8)`, and `rctl(8)`
commands used to manage my jail-based virtualization platform on FreeBSD.

`jailctl` isn't a general-purpose tool. It is tailored to my opinionated methods
for managing jails. Specifically:

- Everything uses ZFS.

- Every jail is a thick VNET jail.

- There is *zero* IPv6 support.

- All jails are started automatically on boot.

- Each jail's configuration lives at /etc/jails.conf.d/$jailname.conf.

- Jail resource limits are managed by `rctl(8)`.

- Each jail gets two ZFS datasets: `$jail/os` and `$jail/data`.

- The 'os' dataset contains the root filesystem. It is cloned from a ZFS
  template and is managed by the host.

- The 'data' dataset contains arbitrary data. It is delegated to the jail
  and managed from within the jail itself.

  This allows us to wipe and reprovision the OS from a fresh template,
  while leaving application data intact.

This tool does not touch your networking configuration, other than to create
an epair(4) virtual ethernet device for each running jail. If you need
bridges, LAGGs, or VLAN interfaces, you'll have to configure those yourself.

### Usage

Initialize the jail dataset:

    $ jailctl init

The previous command will automatically create a template dataset with a FreeBSD
rootfs:

    $ jailctl list-templates
    freebsd13

Let's create a new jail based on the template. The jail will be bridged to the
physical network interface `em0`. Since we don't specify an IP address, DHCP
will be used.

    $ jailctl create -i em0 test1 freebsd13

Our `test1` jail should now be running:

    $ jailctl list
    JAIL   STATUS
    test1  running

We can run a command within the jail using `jailctl exec`:

    $ jailctl exec test1 uname -a
    FreeBSD test1.example.com 13.2-RELEASE FreeBSD 13.2-RELEASE releng/13.2-n254617-525ecfdad597 GENERIC amd64

Or get a shell within the jail using `jailctl shell`:

    $ jailctl shell test1
    # echo "I'm in jail: $(hostname)"
    I'm in jail: test1.example.com

Let's create another jail, this time with a static IP and resource limits. We'll
use a ZFS quota of 128 GB, a memory limit of 4 GB, and a CPU limit of 400.

The CPU limit is given in terms of percentage per core. A limit of 400 means that
our jail can hog at most 4 CPU cores.

    $ jailctl create \
       -i em0 \
       -a 10.11.199.17 -n 255.255.255.0 -g 10.11.199.1 \
       -r 8.8.8.8 -r 8.8.4.4 \
       -s sub.example.com -s example.com \
       -c 400 -m 4G -q 128G
       test2 freebsd13

Let's verify our network configuration:

    $ jailctl exec test2 ifconfig
    jail0: flags=8863<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> metric 0 mtu 1500
            options=8<VLAN_MTU>
            ether 06:49:5a:c0:73:01
            hwaddr 02:63:8c:69:a0:0b
            inet 10.11.199.17 netmask 0xffffff00 broadcast 10.11.199.255
            groups: epair
            media: Ethernet 10Gbase-T (10Gbase-T <full-duplex>)
            status: active
            nd6 options=29<PERFORMNUD,IFDISABLED,AUTO_LINKLOCAL>

    $ jailctl exec test2 cat /etc/resolv.conf
    nameserver 8.8.8.8
    nameserver 8.8.4.4
    search sub.example.com example.com

We can also inspect various runtime information about a jail using `jailctl statistics`:

    $ jailctl statistics test2
    ---------------------------- JAIL STATUS -----------------------------
    jid  name   path                    osrelease     host.hostname
    4    test2  /usr/local/jails/test2  13.2-RELEASE  test2.example.com

    ---------------------------- ZFS DATASET -----------------------------
    NAME                    QUOTA   USED  AVAIL  MOUNTPOINT
    zroot/jails/test2/data   128G    96K   128G  none
    zroot/jails/test2/os      24G   288K  24.0G  /usr/local/jails/test2

    --------------------------- RESOURCE LIMITS --------------------------
    vmemoryuse:deny=4096M
    pcpu:deny=400

    --------------------------- RESOURCE USAGE ---------------------------
    memoryuse  maxproc  openfiles  vmemoryuse  swapuse  pcpu  readbps  writebps  readiops  writeiops
    5192K      2        40         25M         0        0     0        0         0         0

    ----------------------------- PROCESSES ------------------------------
    USER  PID %CPU %MEM   VSZ  RSS TT  STAT STARTED    TIME COMMAND
    root 4056  0.0  0.0 12868 2732  -  SsJ  14:09   0:00.00 /usr/sbin/syslogd -ss
    root 4095  0.0  0.0 12908 2492  -  IsJ  14:09   0:00.00 /usr/sbin/cron -s

Delete a jail with `jailctl destroy`:

    $ jailctl destroy test2
    Really delete test2? (y/N) y
    test2: run command in jail as root: /bin/sh /etc/rc.shutdown
    Stopping cron.
    Waiting for PIDS: 4095.
    .
    Terminated
    test2: sent SIGTERM to: 4056
    test2: removed
    test2: run command as root: jib destroy 9ef9066971f
    test2: run command as root: /sbin/umount /usr/local/jails/test2/dev
    /etc/rc.conf: jail_list: test1 test2 -> test1
    will destroy zroot/jails/test2/os
    will destroy zroot/jails/test2/data
    will destroy zroot/jails/test2
