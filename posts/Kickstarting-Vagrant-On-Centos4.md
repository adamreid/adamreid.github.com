<!-- 
.. title: Kickstarting a CentOS / RedHat 4 Vagrant Image
.. slug: Kickstarting-Vagrant-On-Centos4
.. date: 2012/05/21 00:00:00
.. tags: puppet centos vagrant
.. link: adamreid.ca/2012/05/21/Kickstarting-Vagrant-On-Centos4/
.. description: My method of bootstrapping an Enterprise Linux system for use with puppet and vagrant
-->

In my [previous post][es4-puppet-kickstart] I explained one way to get [puppet][puppet] running on CentOS 4 / RedHat ES-4 using a kickstart. Building from that kickstart, I am going to show how to set up a [VirtualBox][VirtualBox] VM suitable for use with [Vagrant][Vagrant]. Although I focus on a dated distribution here, these instructions can be applied to other distributions pretty easily. The kickstart can only be used by distributions that support kickstart installations. That means any RedHat or CentOS and possibly others with modifications.

If you don't know why you might want to use [Vagrant][Vagrant] I will briefly explain. Vagrant is a tool that combines virtualization via VirtualBox with configuration management via [Puppet][Puppet] and/or [Chef][Chef] to let you create reproducible development environments. You write a `Vagrantfile` that tells vagrant what your machine should look like and what puppet/chef configurations to apply. This means you don't need to keep a lot of VM's around when they aren't being used. Instead you can re-create the machine with vagrant when you need it, and destroy it when you're done. It has some nice features such as automatic ssh forwarding, optional port forwarding of other services (http for example), and automatic folder sharing. This last one is really nice, as it lets you write an application on your PC with whatever tools you would normally use and automatically have the code available in a VM for testing.

In this kickstart I don't provide Chef, although it can be added with `gem install chef --no-doc --no-ri` if you would like it to be included in your images.

The set up
----------

For this to work you are going to need the following things on your PC. I will not go into the details of installing VirtualBox or Vagrant, they both have excellent documentation available.

* [VirtualBox]
* [Vagrant]
* A terminal
* A CentOS / RedHat 4 DVD iso
* A web server, I use Python 2.7 and `python -m 'http.server'`

When VirtualBox is installed, copy (or link), the `VBoxGuestAdditions.iso` file into a directory to work in. For linux hosts you can usually find it in `/usr/lib/virtualbox/additions`, for Mac OS hosts it will be in `/Applications/VirtualBox.app/MacOS/`. Grab a copy of the latest [kickstart][kickstart] from github. Serve these files via http using your favourite web server. I
use Python and it's `http.server` module for Python 2.7 or `SimpleHTTPServer` for versions < 2.7.

~~~ {bash}
areid $ cd ~/kickstarts/puppet
puppet $ ln -s /usr/lib/virtualbox/additions/VBoxGuestAdditions.iso .
puppet $ wget https://raw.github.com/adamreid/kickstarts/master/puppet-es4-vagrant-ks.cfg
puppet $ python -m 'http.server'
Serving HTTP on 0.0.0.0 port 8000 ...
~~~

The kickstart
-------------

This kickstart is an extension of the [puppet-es4][es4-puppet-kickstart] with the addition of a few steps to ready it for [vagrant][Vagrant] and the disabling of a few services that I would consider unnecessary for the VM's intended purpose.

Most changes here are to conform to the conventions outlined in the [Vagrant base box][base-box] documentation. Specifically:
* The root password is changed to vagrant.
* The vagrant user is added to the admin group.
* The admin group is given passwordless sudo access.
* The vagrant user's `.ssh/authorized_keys` file contains a known (insecure) key for pubkey authentication.
* The VirtualBox guest additions are installed.

A final step is then run to help keep the size of the exported vagrant box down. A file is created in `/tmp` and filled with 0's until the disk runs out of space, then is deleted. This step is optional. If you are doing repeated testing of the kickstart, this step should be commented out to save time.

~~~ {bash}
# Kickstart file for Puppet on CentOS 4/RedHat ES-4
# Tested on ES-4 U8 / CentOS 4.8
# May work on older enterprise 4 systems.
#
# Adam Reid
# http://github.com/adamreid/kickstarts/puppet-es4-ks.cfg

install
cdrom
lang en_US.UTF-8
langsupport --default=en_US.UTF-8 en_US.UTF-8
keyboard us
xconfig --card "VESA driver (generic)" --videoram 12288 --hsync 31.5-37.9 --vsync 50-70 --resolution 800x600 --depth 16
network --device eth0 --bootproto dhcp
rootpw vagrant
firewall --disabled
selinux --disabled
authconfig --enableshadow --passalgo=md5
timezone --utc America/New_York
bootloader --location=mbr
# The following is the partition information you requested
# Note that any partitions you deleted are not expressed
# here so unless you clear all partitions first, this is
# not guaranteed to work
clearpart --all --drives=sda
part /boot --fstype ext3 --size=100 --ondisk=sda
part pv.2 --size=0 --grow --ondisk=sda
volgroup VolGroup00 --pesize=32768 pv.2
logvol swap --fstype swap --name=LogVol01 --vgname=VolGroup00 --size=512 --grow --maxsize=1024
logvol / --fstype ext3 --name=LogVol00 --vgname=VolGroup00 --size=1024 --grow

%packages
lvm2
kernel
e2fsprogs
grub

# required to build ruby
# should be kept for gems that build native extensions.
automake
gcc
cpp
glibc-devel
glibc-headers
glibc-kernheaders
glibc
glibc-common
libgcc

# required to build ruby bindings, can be removed after
zlib-devel
openssl-devel
readline-devel

%post

# Change to a vt to see progress

exec < /dev/tty3 > /dev/tty3
chvt 3

# redirect output to ks-post.log including stdout and stderr
(
	#######################################################
	# Build Ruby
	#######################################################

	# Keep it clean
	mkdir /tmp/ruby
	cd /tmp/ruby

	# autoconf 2.60 is required to build ruby
	wget http://ftp.gnu.org/gnu/autoconf/autoconf-2.60.tar.gz
	tar -xzf autoconf-2.60.tar.gz
	cd autoconf-2.60
	./configure --prefix=/usr && make && make install
	cd /tmp/ruby

	# build ruby-1.8.7-p358
	wget http://ftp.ruby-lang.org/pub/ruby/1.8/ruby-1.8.7-p358.tar.bz2
	tar -xjf ruby-1.8.7-p358.tar.bz2
	cd ruby-1.8.7-p358
	autoconf
	./configure --prefix=/usr && make && make install
	cd /tmp/ruby

	# install ruby-gems 1.8.10
	wget http://production.cf.rubygems.org/rubygems/rubygems-1.8.10.tgz
	tar -xzf rubygems-1.8.10.tgz
	cd rubygems-1.8.10
	/usr/bin/ruby setup.rb


	# clean up
	cd /
	rm -rf /tmp/ruby

	#######################################################
	# Install Puppet
	#######################################################
	gem install puppet --no-rdoc --no-ri

	# add the puppet group
	groupadd puppet

	#######################################################
	# Install VirtualBox Guest Additions
	#
	# Note: You will need to provide a copy of the
	# VirtualBoX Guest Addititons iso on a web server.
	#######################################################
	cd /tmp
	wget http://192.168.1.3:8000/VBoxGuestAdditions.iso
	mkdir /tmp/isomount
	mount -t iso9660 -o loop /tmp/VBoxGuestAdditions.iso /tmp/isomount
	
	/tmp/isomount/VBoxLinuxAdditions.run
	umount isomount
	rm VBoxGuestAdditions.iso

	#######################################################
	# Turn off un-needed services
	#######################################################
	chkconfig sendmail off
	chkconfig vbox-add-x11 off
	chkconfig smartd off
	chkconfig ntpd off
	chkconfig cupsd off

	#######################################################
	# Setup for Vagrant
	#######################################################
	groupadd admin
	useradd -g admin vagrant
	echo 'Defaults env_keep="SSH_AUTH_SOCK"' >> /etc/sudoers
	echo '%admin	ALL=NOPASSWD: ALL' >> /etc/sudoers

	# Add vagrant insecure private key for key auth
	# Make your own if this is private.
	# See http://vagrantup.com/docs/base_boxes.html
	mkdir /home/vagrant/.ssh
	echo "ssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAQEA6NF8iallvQVp22WDkTkyrtvp9eWW6A8YVr+kz4TjGYe7gHzIw+niNltGEFHzD8+v1I2YJ6oXevct1YeS0o9HZyN1Q9qgCgzUFtdOKLv6IedplqoPkcmF0aYet2PkEDo3MlTBckFXPITAMzF8dJSIFo9D8HfdOV0IAdx4O7PtixWKn5y2hMNG0zQPyUecp4pzC6kivAIhyfHilFR61RGL+GPXQ2MWZWFYbAGjyiYJnAmCP3NOTd0jMZEnDkbUvxhMmBYSdETk1rRgm+R4LOzFUGaHqHDLKLX+FIPKcF96hrucXzcWyLbIbEgE98OHlnVYCzRdK8jlqm8tehUc9c9WhQ== vagrant insecure public key" > /home/vagrant/.ssh/authorized_keys

	# Clean up unused disk space so compressed image is smaller.
	cat /dev/zero > /tmp/zero.fill
	rm /tmp/zero.fill

	#######################################################
	# The system can now be packaged with 
	# `vagrant package VMNAME`
	#######################################################
	echo 'You can now package this box with `vagrant package VMNAME`'

) 2>&1 | /usr/bin/tee /root/ks-post.log

# switch back to gui
chvt 7

~~~

Creating the Vagrant Image
--------------------------

Now that the kickstart has been installed it is ready to be packaged as a vagrant base box.

I named my base image *vagrant-es4* in VirtualBox, so when following this example make sure to replace my VM name with yours.

~~~ {bash}
puppet $ vagrant package --base vagrant-es4
puppet $ vagrant box add es4base package.box
~~~

The image is now ready for you to use in your own vagrant images. Create a new vagrant configuration to test the base image with.

~~~ {bash}
puppet $ cd ~/vms
vms $ mkdir es4test
vms $ cd es4test
vms $ vagrant init es4base
vms $ vagrant up
vms $ vagrant ssh
~~~

The last command `vagrant ssh` gives you shell access to the virtual machine you just created. If you look in the machines `/vagrant` directory you will notice that your `Vagrantfile` is there. This is a VirtualBox "shared folder" that is provided for you by vagrant. The directory it's sharing is the same directory that `vagrant init` was run inside of in the steps above. 

When developing an application you can see how this little convenience will make a big difference in testing your application in a virtual environment. You just create a `Vagrantfile` with the configuration you require, spin up the VM, and run your application from the shared folder. Keeping the `Vagrantfile` in source with your application will remind you to keep it up to date as the applications requirements change, and makes it easy for new developers to quickly set up a known-good environment to test in.

[Puppet]: http://puppetlabs.com/
[Chef]: http://www.opscode.com/chef/
[CentOS]: http://www.centos.org/
[RedHat]: http://www.redhat.com/
[Vagrant]: http://vagrantup.com/
[base-box]: http://vagrantup.com/docs/base_boxes.html
[VirtualBox]: https://www.virtualbox.org/
[es4-puppet-kickstart]: http://adamreid.ca/2012/05/15/Kickstarting-Puppet-On-Centos4/
[kickstart]: https://raw.github.com/adamreid/kickstarts/master/puppet-es4-vagrant-ks.cfg