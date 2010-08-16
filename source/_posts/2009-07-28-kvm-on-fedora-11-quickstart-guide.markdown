---
title: KVM on Fedora 11 QuickStart Guide
---
With the upcoming release of RHEL 5.4, Red Hat officially enters the
virtualization fray with a KVM-based solution. Although KVM is exciting for
several reasons, e.g. it's free, fully supported by the Linux kernel community,
doesn't require any special hardware past the increasingly common Intel-VT or
AMD-V extensions, etc, its also represents the next stage of virtualization evolution.

I'll expound in a future post about how and why (admittedly with the benefit of
hindsight) KVM represents a far more elegant model for virtualization, but
after the jump today you'll find a minimal set of instructions to rapidly get a
Debian Lenny virtual guest up and running on a fresh out-of-the-box Fedora 11
server installation.

Lengthy instructions on Fedora virtualization suitable for KVM and/or Xen can
be found at
[the Fedora wiki](http://fedoraproject.org/wiki/Virtualization_Quick_Start#Using_virtualization_on_fedora)
The following lists the minimal set of steps to get a guest running focusing
soley on KVM to the exclusion of Xen.

Our setup comprises a freshly installed remote Fedora 11 server serving as the
KVM target (bash prompt _server_) and a local Fedora 11 laptop (bash prompt _laptop_).

First we'll shell into the remote server to install and configure all the
necessary KVM software. Verify we're using Fedora 11 with a Intel-VT or AMD-V
processor:

    [root@server ~]# uname -a && egrep '(vmx|svm)' /proc/cpuinfo | wc -l
    Linux server 2.6.29.4-167.fc11.x86_64 #1 SMP Wed May 27 17:27:08 EDT 2009 x86_64 x86_64 x86_64 GNU/Linux
    4

This server is running the modified 2.6.29 Linux kernel that comes stock with
Fedora Core 11. It possesses 4 processing cores enabled with Intel-VT. Installing
the basic tools required for KVM is our first task, luckily Fedora provides a
convenient meta-package:

    [root@server ~]# yum groupinstall 'Virtualization'

Alongside kvm/qemu, the Virtualization group also includes the
[libvirt](http://libvirt.org/) toolset. Libvirt provides a common abstraction
across several types of hypervisors. This abstraction includes simplified
installation and remote graphical console access a la $VMW's VirtualCenter. To
access these features we must start the libvirtd daemon.

    [root@server ~]# /etc/init.d/libvirtd start
    Starting libvirtd daemon:                                  [  OK  ]

Using the _virsh_ command line tool, verify the currently guestless system is
running and ready to accept commands:

    [root@server ~]# virsh -c qemu:///system list
    Id Name                 State
    ----------------------------------

Now we're ready to install our guest. We'll need to download an ISO image to
install our virtual machine. To keep things simple we'll go with Debian Stable
aka _Lenny_ (as of this writing). We'll create a top-level directory to store
both the ISO image and the virtual machine disk image:

    [root@server ~]# mkdir /kvm && cd /kvm
    [root@server kvm]# wget http://cdimage.debian.org/debian-cd/5.0.2/amd64/iso-cd/debian-502-amd64-netinst.iso
    ...
    [snip]
    ...
    Saving to: `debian-502-amd64-netinst.iso.1'
    100%[==================================================================================================>] 137,713,664 1.49M/s in 93s
    2009-07-27 19:10:24 (1.56 MB/s) - `debian-502-amd64-netinst.iso' saved [137713664/137713664]

Now armed with the KVM/libvirt tools and an ISO image for installation, we issue
a single _virt-install_ command to a) create a backing disk image and b) boot
a newly created virtual guest off of the ISO:

    root@server kvm]# virt-install --connect qemu:///system -n lenny0 \
    > -r 512 --disk path=/kvm/lenny0.qcow2,size=16 \
    > -c /kvm/debian-502-amd64-netinst.iso --noautoconsole --os-type linux \
    > --os-variant debianlenny --accelerate --hvm

    Starting install...
    Creating storage file...  |  16 GB     00:00
    Creating domain...        |    0 B     00:01
    Domain installation still in progress. You can reconnect to the console to complete the installation process.

The "domain" referred to in virt-install's output is our new guest. We've
created a single-core virtual server named lenny0 with 512 MB of RAM and a 16GB
virtual disk housed on server at /kvm/lenny0.qcow2. Furthermore, lenny0 has been
booted off of our Debian net-install ISO image.

Now we can use the graphical _virt-manager_ tool locally from our laptop to
connect to the virtual server's graphical console (basically VNC) and kick off
the Debian net-install process. In theory we should be able to run virt-manager
locally on our laptop and connect over ssh to the libvirtd daemon on the remote
server. In practice it's still pretty shaky and has some onerous requirements.
Notably that there's no progress indicator informing how much longer remains
before the connection initializes (and it takes a long time). So if you want
to try it out, [good luck](http://virt-manager.org/page/RemoteSSH"), YMMV
(hopefully someone at $RHAT puts some polish on this aspect soon).

I've personally chosen to avoid the whole debacle and just run virt-manager on
the remote server and use X Forwarding over ssh to interact with it from the
laptop. It's slow, but sufficient for Debian's curses-based installer
(I actually did this across the continental US).

    fedora:lm jruscio$ ssh -X root@server 'virt-manager' &amp;
    [1] 20038

The virt-manager tool should show a single VM, lenny0, running on server.
Double-clicking on the lenny0 entry brings up the VNC console, that should show
the Debian net-install splash screen. Start the installation and for the most
part select all the defaults (I removed _Desktop_ from the additional packages).

After the installer finishes it leaves the machine in a halted state. Boot the
machine with the _Run_ button above the VNC console and log in as root. As a
final step install ssh and use ifconfig to determine the IP address DHCP granted
the virtual server:

    lenny0:~# apt-get install ssh
    [snip]
    lenny0:~# ifconfig eth0 |grep inet
	      inet addr:192.168.122.99  Bcast:192.168.122.255  Mask:255.255.255.0
	      inet6 addr: fe80::5652:ff:fe46:6f93/64 Scope:Link

That was pretty easy and we're now able to shell into our virtual guest from
the physical server hosting it!:
    server:~ jruscio$ ssh 192.168.122.99
    jruscio@192.168.122.99's password:
    Linux lenny0 2.6.26-2-amd64 #1 SMP Sun Jun 21 04:47:08 UTC 2009 x86_64
