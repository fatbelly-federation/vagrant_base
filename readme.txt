# install needed packages
yum install -y wget kernel-devel gcc make kernel-devel-$( uname -r )

# install vagrant
wget -O /tmp/vagrant.rpm https://releases.hashicorp.com/vagrant/2.1.2/vagrant_2.1.2_x86_64.rpm && yum install -y /tmp/vagrant.rpm

# install virtualbox
# find your package at --> https://www.virtualbox.org/wiki/Linux_Downloads
wget -O /tmp/virtualbox.rpm https://download.virtualbox.org/virtualbox/5.2.16/VirtualBox-5.2-5.2.16_123759_el7-1.x86_64.rpm && yum install -y /tmp/virtualbox.rpm

# build & configure kernel modules
/sbin/vboxconfig

# add sshfs plugin
vagrant plugin install vagrant-sshfs

## now all the needed bits should be installed
## NOTE: you can't use Virtualbox in AWS because EC2 instances are missing the vmx cpu feature

## there is a aws provider for vagrant
## https://github.com/mitchellh/vagrant-aws
