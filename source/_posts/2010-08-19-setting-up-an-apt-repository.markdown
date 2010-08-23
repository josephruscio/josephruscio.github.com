---
title: Setting up an APT Repository
---

I recently set out to create an APT repository to host publicly available
packages for [Librato's Silverline](http://librato.com). Our users install
a tiny agent on their servers and a hosted APT repository provides those
using Debian/Ubuntu a configuration management solution superior to
manual downloads.

Our APT repository must support multiple distributions of both Debian
(e.g. Etch, Lenny) and it's popular derivative Ubuntu (e.g. Karmic, Lucid).
The repository must support both the i386 and x86_64 architectures across
all such distributions. A short google search reveals that `reprepro` is
a tool designed for just such a purpose. There are also
[informative](http://www.jejik.com/articles/2006/09/setting_up_and_managing_an_apt_repository_with_reprepro/)
[reprepro](http://www.danielbond.org/archives/114) [walkthroughs](http://davehall.com.au/blog/dave/2010/02/06/howto-setup-private-package-repository-reprepro-nginx)
already online, but they all gloss over different subtle yet extremely
important details.

So what follows is my attempt to minimally but comprehensively document the
procedure assuming a familiarity with shell basics, but little or no experience
with Debian packging past `apt-get update/upgrade/install`.

### Host Environment

Our APT repository is hosted on a fresh installation of Ubuntu 10.04 Lucid, but
the procedure should work on any recent version of Debian/Ubuntu.

### Package Configuration

The most important detail that AFAIK isn't covered in any of the tutorials
had to do with package naming conventions. The naive assumption (at least
on my part) is that you'll have a different build of your package for
each distro/arch combination, and import them into your repository as such.
In other words reprepro should track the distro/arch of each import.
In actuality, each build's `<PACKAGE>_<VERSION>_<ARCH>` must be unique, even though
you specify the distro during the `includedeb` operation.

To address this requirement there's a common practice of appending the
distro to the package version e.g. `1.0.7` becomes `1.0.7~lenny1` or `1.0.7+etch4`.
(There doesn't appear to be a common consensus on the joining character.) Adding
the second version number after the distro string versions the
package itself, enabling updates that only contain changes to the packaging itself.

If you do not build your packages to include the distro as part of the version, you
will not be able to import a package for more than one distro. You can verify your
package was built correctly by inspecting the `Version` field listed by `dpkg -I`:
    ubuntu:~/lenny$ dpkg -I librato_silverline_debian_lenny_5.0.3.x86_64.deb
     new debian package, version 2.0.
     size 306916 bytes: control archive= 834 bytes.
	 255 bytes,     7 lines      conffiles            
	 277 bytes,    10 lines      control              
	 991 bytes,    26 lines   *  postinst             #!/bin/sh
     Package: librato-silverline
     Version: 2.0.7~lenny
    ...

### GnuPG

Assuming you want sign your debian packages to create a secure APT repository,
you need a GPG key. If you already have a GPG key you can skip to the next
section. Otherwise first ensure that GPG is installed:
    ubuntu:~$ sudo apt-get install gnupg

Create a key with the `--gen-key` command. Select the defaults and be sure to
record the key's passphrase, you'll need it each time you import a package into
the repository:
    ubuntu:~$ gpg --gen-key

### Install and Configure reprepro

Armed with properly configured packages and a GPG key with which to sign them,
we're now ready to start constructing the repository itself. Start by installing
`reprepro`:
    ubuntu:-$ sudo apt-get install reprepro

Create a directory to serve as the _base_ of the Debian repository. In our setup
`/var/packages/debian` serves as the Debian repository base. In the future
`/var/packages/ubuntu` will house the Ubuntu repository base. We'll change ownership
of these directories to the `ubuntu` user that owns the signing key.
    ubuntu:~$ sudo mkdir /var/packages
    ubuntu:~$ sudo chown ubuntu:ubuntu /var/packages
    ubuntu:~$ mkdir /var/packages/debian

Each repository needs a top-level `conf` directory:
    ubuntu:~$ mkdir /var/packages/debian/conf

Create a file named `distributions` in the `conf` directory. You'll add a set
of newline separated configuration blocks (one for each supported distro) to
this file. In this example the repository supports `etch` and `lenny`. In the future
`squeeze` support could be added with a third block:
    Origin: Librato, Inc.
    Label: Librato, Inc.
    Codename: etch
    Architectures: i386 amd64
    Components: non-free
    Description: Librato APT Repository
    SignWith: yes
    DebOverride: override.etch
    DscOverride: override.etch

    Origin: Librato, Inc.
    Label: Librato, Inc.
    Codename: lenny
    Architectures: i386 amd64
    Components: non-free
    Description: Librato APT Repository
    SignWith: yes
    DebOverride: override.lenny
    DscOverride: override.lenny

Fill in the `Origin`, `Label`, and `Description` as you see fit. `Codename` identifies the
distro the block describes and `Architectures` is self-explanatory (note that `sources` is a valid arch).
`Components` lists the component of the packages in the repository e.g. for
Debian `main`, `contrib`, or `non-free`. `DebOverride` and `DscOverride` exceed the scope of
this posting, the reader may either research them on their own or set them as shown.

`SignWith` instructs reprepro that these packages should be signed. The `yes`
is sufficient as the `ubuntu` user only has the one key generated above. If
you have more than one key you can specify the ID of the signing key with this
field.

Create empty override files:
    ubuntu:~$ touch /var/packages/conf/override.etch
    ubuntu:~$ touch /var/packages/conf/override.lenny

Create a file named `options` in the `conf` directory and fill it with the
following content. This file will store a
set of options for reprepro to always run. We only use a few but there are
more available on the `reprepro` manpage.
    verbose
    ask-passphrase
    basedir .

The `verbose` option is self-explanatory. The ask-passphrase option instructs
`reprepro` to ask us for a GPG key passphrase during the import (otherwise
it'll fail to sign the packages). The `basedir .` implies that we'll be
running all of our `reprepro` commands with the repository base as our
working directory:
    ubuntu:~$ cd /var/packages/debian

### Import Packages

The repository base is completely configured and ready for package importing. Just
use the `includedeb` command and specify the correct distro. In our case:
    ubuntu:/var/packages/debian$ reprepro includedeb etch ~/librato_silverline_debian_etch_4.0r8.x86_64.deb
    ...
    ubuntu:/var/packages/debian$ reprepro includedeb lenny ~/ubuntu/librato_silverline_debian_lenny_5.0.3.x86_64.deb

Each `reprepro includedeb` operation should prompt for your GPG key passphrase and import the package.

### Repository Access

We now have a proper Debian repository and all that remains is making it
accessible for installations over the WAN. We just need to serve static
files so we'll use `nginx` to serve the packages over HTTP.
    ubuntu:/var/packages/debian$ sudo apt-get install nginx

Configure the APT server in `/etc/nginx/sites-available/vhost-packages.conf`:
    server {
      listen 80;
      server_name apt.librato.com;

      access_log /var/log/nginx/packages-error.log;
      error_log /var/log/nginx/packages-error.log;

      location / {
	root /var/packages;
	index index.html;
      }

      location ~ /(.*)/conf {
	deny all;
      }

      location ~ /(.*)/db {
	deny all;
      }
    }

Configure the hash bucket size by creating the file
`/etc/nginx/conf.d/server_names_hash_bucket_size.conf`
    server_names_hash_bucket_size 64;

Enable the APT server:
    ubuntu:/var/packages/debian$ cd /etc/nginx/sites-enabled
    ubuntu:/etc/nginx/sites-enabled$ sudo ln -s ../sites-available/vhosts-packages.conf .
    ubuntu:/etc/nginx/sites-enabled$ sudo service nginx start

### Enable Public Key Access
Our users will need our public key to validate the signed packages we're
providing. Placing it in the root of our `nginx` configuration above
makes it accessible with a simple `curl` invocation:
    gpg --armor --output /var/packages/packages.librato.key --export apt@librato.com

### Installation Test
Now we can test our fully operational APT repository. On some
candidate machine log in as root and add our repository to
`/etc/apt/sources.list`:
    deb http://apt.librato.com/debian/ lenny non-free

Import the repository's public key:
    lenny:# curl http://apt.librato.com/packages.librato.key | apt-key add -

Fetch the list of packages available at the new source:
    lenny:# apt-get update

Install the hosted package!
    lenny:# apt-get install librato-silverline
