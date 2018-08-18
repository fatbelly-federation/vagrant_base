# -*- mode: ruby -*-
# vi: set ft=ruby :

# Vagrantfile API/syntax version. Don't touch unless you know what you're doing!
VAGRANTFILE_API_VERSION = "2"


# Load in some Vagrant settings from a yaml file
require 'yaml'
settings = YAML.load_file 'settings.yml'

puppet_options = '--summarize --reports store'

# this picks which box to load & run
box = settings['boxes'][settings['os_version']]

# define the hardening script to use
hardening_script = 'scripts/'+ box['hardening_script']

# output some useful info(which box we're going to run)
# and its source url
puts 'Box: '+ box['name']
if !box['url'].nil?
    puts 'URL: ' + box['url']
end

# Allow the user to turn puppet provisioning off and on
if settings['puppet_enabled']

    if settings['puppet_debug']
            puppet_options += ' --debug'
    end

    if settings['puppet_verbose']
            puppet_options += ' --verbose'
    end
end

# Disable deprecated warnings if enabled
if settings['puppet_deprecations']
  puppet_options += ' --disable_warnings deprecations'
end

# Configure the vagrant instance
Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|

  config.vm.box = box['name']
  config.vm.box_version = box['version']
  if !box['url'].nil?
      config.vm.box_url = box['url']
  end

  # first try to pull name from environment
  if !ENV["VAGRANT_NAME"].nil?
      config.vm.host_name = ENV["VAGRANT_NAME"]
  elsif !settings['host_name'].nil?
      # pull hostname from the yaml config
      config.vm.host_name = settings['host_name']
  else
      # unable to find a name for our vagrant, so we'll go with a default name
      config.vm.host_name = "beeblebrox"
  end

  # set the ip address of the guest
  config.vm.network "private_network", ip: settings['ip_address']

  # define some useful port forwarding
  config.vm.network "forwarded_port", guest: 80, host: 8080
  config.vm.network "forwarded_port", guest: 443, host: 8443

  # how much memory should the guest have
  config.vm.provider "virtualbox" do |vb|
    if !settings['guest_memory'].nil?
        vb.memory = settings['guest_memory']
    else
        # no memory setting defined, so we'll default to 512mb
        vb.memory = "512"
    end

    # how many cpu?
    if !settings['cpus'].nil?
        vb.cpus = settings['cpus']
    else
        # nothing explictly set, so default to two cpu
        vb.cpus = 2
    end

    # setup nat for dns resolution
    vb.customize ["modifyvm", :id, "--natdnsproxy1", "on"]
    vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
  end

  # apply system hardening script if option selected
  if settings['system_hardening']
    config.vm.provision "shell" do |s|
      s.name = "hardening"
      s.path = hardening_script
    end
  end

  # bootstrap the system
  # set & pass environment to the script
  # ref: https://stackoverflow.com/questions/19648088/pass-environment-variables-to-vagrant-shell-provisioner
  config.vm.provision "shell" do |s|
    s.name = "bootstrap"
    s.path = "scripts/provision.sh"
    s.env = {"VAR_EXAMPLE" => "silly-value"}
  end

  # install puppet if necessary
  if settings['puppet_install']
    config.vm.provision "shell" do |s|
      s.name = "puppet-install"
      s.path = "scripts/puppet-install.sh"
      s.args = ['--install', box['puppet_rpm'], box['puppet_agent_version']]
    end
  end

  # default shared folders
  # install virtualbox extensions if necessary
  # sshfs seems a bit more stable
  # virtualbox guest add-ons keeps installing bad symlink for vboxsf
  # https://www.virtualbox.org/ticket/9307
  # http://stackoverflow.com/questions/22717428/vagrant-error-failed-to-mount-folders-in-linux-guest/43598079#43598079
  # sshfs plugins --> https://github.com/dustymabe/vagrant-sshfs

  # mount our local directory to /vagrant
  # this keeps vagrant from using rsync to populate /vagrant within the guest
  config.vm.synced_folder ".", "/vagrant", type:"sshfs"

  # mount the puppet folder to /etc/puppet in the guest
  if !ENV["PUPPET_DIR"].nil?
      # found an environment value, so we'll use it
      config.vm.synced_folder ENV["PUPPET_DIR"], "/etc/puppet", type:"sshfs"
  elsif !settings['puppet_dir'].nil?
      # use the puppet_dir value from our settings.yml
      config.vm.synced_folder settings['puppet_dir'], "/etc/puppet", type:"sshfs"
  else
      # no explict value set, so use the puppet directory in our current directory
      config.vm.synced_folder "puppet/", "/etc/puppet", type:"sshfs"
  end

  # mount the inspec folder to /inspec in the guest
  if !ENV["INSPEC_DIR"].nil?
      # found an environment value, so we'll use it
      config.vm.synced_folder ENV["INSPEC_DIR"], "/inspec", type:"sshfs"
  elsif !settings['inspec_dir'].nil?
      # use the inspec_dir value from our settings.yml
      config.vm.synced_folder settings['inspec_dir'], "/inspec", type:"sshfs"
  else
      # no explict value set, so use the inspec directory in our current directory
      config.vm.synced_folder "inspec/", "/inspec", type:"sshfs"
  end

  # configure the puppet provisioner
  if settings['puppet_enabled']
    # setup custom facts for use by puppet
    config.vm.provision "shell" do |s|
        s.name = "puppet-facts"
        s.path = "scripts/facts.sh"
        s.args = ['-r', settings['role'], '-s', settings['stage'], '-p', settings['platform']]
    end

    # this is where vagrant runs `puppet apply` for us
    # ref: https://www.vagrantup.com/docs/provisioning/puppet_apply.html
      config.vm.provision "puppet" do |puppet|
        puppet.options        = puppet_options
        puppet.environment_path = "puppet/environments"
        puppet.environment = "test"
        puppet.manifests_path = "puppet/environments/test/site-modules/controlrepo/manifests"
        puppet.module_path = ["puppet/environments/test/site-modules/controlrepo/site", "puppet/environments/test/site-modules", "puppet/environments/test/modules"]
        puppet.manifest_file = "site.pp"
        puppet.hiera_config_path = "puppet/environments/test/site-modules/controlrepo/hiera.yaml"
        # use sshfs for shared folder access back to the host
        # https://www.vagrantup.com/docs/provisioning/puppet_apply.html#synced_folder_type
        # https://github.com/dustymabe/vagrant-sshfs
        puppet.synced_folder_type = "sshfs"
      end
  end

  # optionally run yum update
  # typically puppet or the hardening script will run yum update
  if settings['run_yum_update']
    config.vm.provision "shell" do |s|
      s.name = "update system packages"
      s.inline = "yum update -y"
    end
  end

# END of the Vagrantfile
end

