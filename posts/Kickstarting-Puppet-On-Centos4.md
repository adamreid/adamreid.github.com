<!-- 
.. title: Kickstarting Puppet on CentOS 4 / ES-4
.. slug: Kickstarting-Puppet-On-Centos4
.. date: 2012/05/15 00:00:00
.. tags: puppet centos
.. link: http://adamreid.ca/2012/05/21/Kickstarting-Vagrant-On-Centos4/
.. description: My method of bootstrapping an Enterprise Linux system for use with puppet
-->

This is a short post on how I got [Puppet][Puppet] working on [CentOS][CentOS] 4.8. Why would I want to do such a thing when there are pre-packaged RPMs available for Centos 5 or 6? Well the company I work for has a massive number of [RedHat][RedHat] ES-4 U8 (RedHat ES-4 U8 ~ CentOS 4.8) systems deployed all over the world. I work as part of the "Software Build and Configuration" team and we are responsible for setting up systems for QA. Since we use ES-4U8 in production, we must use the same release in QA.

One of the goals I've set for myself in my job is to make these QA systems disposable. When QA test these systems they sometimes require specific configuration or data changes to make sure feature X is working and doesn't break feature Y. You can't very well dispose of a system that takes you 1 or 2 days to manually configure, and that's where puppet comes in.

Puppet (and Chef) allow you to describe the packages, attributes, and configurations that make up a system. They will take a description that you write, called a manifest for puppet and a recipe for chef, and apply changes to a system to make it fit that description. The end result is a reproducible, and therefore disposable, system that can be used for testing a software change.

Unfortunately on a legacy system such as RedHatES-4 the requirements to run puppet out of the box are not met. Fortunately there is not much that needs to be done to set puppet up, and it can easily be accomplished in a kickstart.

The set up
----------

In order to run puppet on any system you need Ruby 1.8.5 or 1.8.7 and rubygems. I chose 1.8.7 for performance improvements, and it's compatibility with puppet and chef. In order to build ruby you will need to manually build autconf 2.60 as CentOS 4.8 ships with autoconf 2.59 which won't work. You will also need to make sure that the following packages are installed in your kickstart environment.

* automake
* gcc
* cpp
* glibc-devel
* glibc-headers
* glibc-kernheaders
* glibc
* glibc-common
* libgcc
* zlib-devel
* openssl-devel
* readline-devel

The kickstart
-------------

This kickstart can be used with CentOS 4 or RedHat 4. It will install a 'minimal system' and the packages required to build Ruby and rubygems. In the `%post` install section it will download and build autoconf, ruby-1.8.7 and rubygems-1.8.10 then use `gem install puppet` to install puppet.

~~~ {bash}
# Kickstart file for Puppet on CentOS 4/RedHat ES-4
# Tested on ES-4 U8 / CentOS 4.8
# May work on older enterprise 4 systems.
#
# May 5, 2012
# Adam Reid

install
cdrom
lang en_US.UTF-8
langsupport --default=en_US.UTF-8 en_US.UTF-8
keyboard us
xconfig --card "VESA driver (generic)" --videoram 12288 --hsync 31.5-37.9 --vsync 50-70 \
	--resolution 800x600 --depth 16
network --device eth0 --bootproto dhcp
# Password = password
rootpw --iscrypted $1$7nqqHQSF$/32jwOnJCOiDivVV7o0iw/
firewall --disabled
selinux --disabled
authconfig --enableshadow --passalgo=md5
timezone --utc America/New_York
bootloader --location=mbr
# The following is the partition information you requested
# Note that any partitions you deleted are not expressed
# here so unless you clear all partitions first, this is
# not guaranteed to work
clearpart --linux --drives=sda
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

%post --log=/root/ks-post.log
# Change to a vt to see progress
chvt 3

# redirect output to ks-post.log including stdout and stderr
(
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

	# install puppet
	gem install puppet
) 2>&1 | tee > /root/ks-post.log

# switch back to gui
chvt 7
~~~

And that's it. That's enough to get you a puppet client system on RedHat ES-4 U8 or CentoOS 4.8. In a later post I will review how to create a [Vagrant][Vagrant] image to test your puppet manifests with.

[Puppet]: http://puppetlabs.com
[CentOS]: http://www.centos.org
[RedHat]: http://www.redhat.com
[Vagrant]: http://vagrantup.com