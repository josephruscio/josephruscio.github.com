---
title: Installing git on CentOS 5.2
---

[CentOS](http://www.centos.org/) for all intents and purposes is a community
supported clone of Red Hat Enterprise Linux (RHEL). If you want to run RHEL
without paying for support, CentOS is as good as it gets. One of the hallmarks
of Enterprise Linux distributions is their rock-steady stability i.e. they only
run really boring, old software. This risk-aversion has a downside in restricted
access to the latest and greatest packages and tools. Case in point, since Red
Hat released RHEL 5.0 in March 2007, Git has really matured from its origins as
a custom kernel development tool into a version control world-beater. Unfortunately
the base CentOS 5.2 repositories don't appear to contain rpms for git. Or at
least the CentOS 5.2 AMI I'm using (ami-e300e68a) doesn't:

    -bash-3.2# yum install git
    Loading "fastestmirror" plugin
    Loading mirror speeds from cached hostfile
    ... [snip] ...
    No package git available.
    Nothing to do

Not to fear, [RPMforge](https://rpmrepo.org/RPMforge") maintains an RPM
repository for CentOS full of goodies, including git (1.5.x as of this writing).
The CentOS Wiki provides
[detailed instructions](http://wiki.centos.org/AdditionalResources/Repositories/RPMForge?action=show&amp;redirect=Repositories%2FRPMForge#head-b06dd43af4eb366c28879a551701b1b5e4aefccd") on configuring RPMforge access, but here's the rapid-fire how-to.

Install _priorities_ to avoid conflicts between packages in RPMforge and the
Base CentOS repositories:

    -bash-3.2# yum install yum-priorities
    ... [snip] ...
    Installed: yum-priorities.noarch 0:1.1.16-13.el5.centos
    Complete!

To configure priorities first edit _/etc/yum.repos.d/CentOS-Base.repo_ and add
a _priority=N_ line to each entry. For [base], [updates], [addons], and
[extras] set _N_ to 1. For [centosplus] set _N_ to 2. Also edit
_/etc/yum.repos.d/rpmforge.repo_ and for [rpmforge] set _N_ > 2.
(I chose 4 because its the next power of 2, and that's how I roll).

Verify our access to RPMforge with a check-update:

    -bash-3.2# yum check-update |grep forge
    * rpmforge: ftp-stud.fht-esslingen.de
    ... [snip] ...

Finally install git:

    -bash-3.2# yum install git
    ... [snip] ...
    Complete!

Enjoy the tasty DVCS goodness!
