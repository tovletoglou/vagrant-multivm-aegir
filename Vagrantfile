# -*- mode: ruby -*-
# vi: set ft=ruby :

# ------------------------------------------------------------------------------
# Variables
# ------------------------------------------------------------------------------

# List with VMs
# Always use the `.dev` suffix.
# Be careful with the private IPs not conflict with other static IPs.
# Add ansible server last in order to boot the target VM first.
vm_array = Array[
  ['aegir.dev', '192.168.33.11'],
  ['ansible.dev', '192.168.33.10']
]

# Personal SSH key, add the path for your id_rsa.pub
# The default targets the home directory.
my_public_ssh_key = "#{Dir.home}/.ssh/id_rsa.pub"

# ------------------------------------------------------------------------------
# Detect if SSH comand exists
# ------------------------------------------------------------------------------

# Cross-platform way of finding an executable in the $PATH.
# More info: http://stackoverflow.com/a/5471032/1692965
def which(cmd)
  exts = ENV['PATHEXT'] ? ENV['PATHEXT'].split(';') : ['']
  ENV['PATH'].split(File::PATH_SEPARATOR).each do |path|
    exts.each do |ext|
      exe = File.join(path, "#{cmd}#{ext}")
      return exe if File.executable?(exe) && !File.directory?(exe)
    end
  end
  nil
end

if which('ssh').to_s == ''
  print "\e[31mCan't find `ssh` command.\nIf you are running on Windows, use `bash` from git for Windows https://git-scm.com/download/win\nand include the `INSTALLATION_PATH\\usr\\bin` on PATH.\e[0m"
  abort
end

if which('ssh-keygen').to_s == ''
  print "\e[31mCan't find `ssh-keygen` command.\nIf you are running on Windows, use `bash` from git for Windows https://git-scm.com/download/win\nand include the `INSTALLATION_PATH\\usr\\bin` on PATH.\e[0m"
  abort
end

# ------------------------------------------------------------------------------
# Generate SSH key
# ------------------------------------------------------------------------------

# Create .ssh directory if don't exists.
Dir.mkdir("#{Dir.pwd}/.ssh") unless File.exist?("#{Dir.pwd}/.ssh")

# If SSH key does not exists (initial boot), then create SSH keys.
unless File.exist?("#{Dir.pwd}/.ssh/id_rsa.pub")
  print "\nThis is the initial run of this Vagrant Multi-VM.\nIt will try to create SSH keys at #{Dir.pwd}/.ssh/\n\n"
  unless system("ssh-keygen -t rsa -b 2048 -f #{Dir.pwd}/.ssh/id_rsa -q -N ''")
    print "\n\e[31mAn error occurred.\e[0m\nTry to genetare the key manually by running the following command in the terminal:\n  ssh-keygen -t rsa -b 2048 -f #{Dir.pwd}/.ssh/id_rsa -q -N ''\n"
    abort
  end
  print "\nSSH key created\n\n"
end

# Create variable with the clients' public key.
if File.exist?("#{Dir.pwd}/.ssh/id_rsa.pub")
  ssh_public_key_project = File.read("#{Dir.pwd}/.ssh/id_rsa.pub")
else
  print "\e[31mNo SSH key found under #{Dir.pwd}/.ssh/id_rsa.pub\e[0m\n"
  abort
end

# Create variable with the hosts's public key.
if File.exist?(my_public_ssh_key)
  ssh_public_key_personal = File.read(my_public_ssh_key)
else
  print "\e[33mNo SSH key found on: #{my_public_ssh_key}\n"
  print "When the VMs will be ready you can access the VMs by:\n"
  print "  vagrant ssh VM_NAME\n"
  print "or\n"
  print "  ssh -i #{Dir.pwd}/.ssh/id_rsa vagrant@VM_URL\e[0m\n\n"
  ssh_public_key_personal = ''
end

# ------------------------------------------------------------------------------
# Vagrant plugins
# ------------------------------------------------------------------------------

unless Vagrant.has_plugin?('vagrant-hostmanager')
  print "\n\e[33mvagrant-hostmanager plugin is not installed\e[0m\nTrying to install it\n"
  unless system('vagrant plugin install vagrant-hostmanager')
    print "\n\e[31mAn error occurred.\e[0m\nTry to install the plugin manually by running the following command in the terminal:\n  vagrant plugin install vagrant-hostmanager\n"
    abort
  end
  print "\nInstall finished successfully. Now run `vagrant #{ARGV[0]}` again.\n"
  abort
end

# ------------------------------------------------------------------------------
# Vagrant configuration
# ------------------------------------------------------------------------------

Vagrant.require_version '>= 1.9.1'

Vagrant.configure('2') do |config|
  # ----------------------------------------------------------------------------
  # Provider configuration (apply to all VMs)
  # ----------------------------------------------------------------------------
  config.vm.provider 'virtualbox' do |vb|
    vb.gui = false
    vb.memory = '1024'
    vb.cpus = 1
  end

  # ----------------------------------------------------------------------------
  # Official Centos 7 box from atlas: https://atlas.hashicorp.com/centos/boxes/7
  # ----------------------------------------------------------------------------
  config.vm.box = 'centos/7'

  # ----------------------------------------------------------------------------
  # Disable shared folders
  # ----------------------------------------------------------------------------
  config.vm.synced_folder '.',
                          '/vagrant',
                          type: 'vboxsf',
                          disabled: true

  # ----------------------------------------------------------------------------
  # Shell provision to restart clients' network (apply to all VMs)
  # ----------------------------------------------------------------------------
  config.vm.provision 'shell',
                      run: 'always',
                      name: 'Restart network (fix Vagrant 1.9.1 bug with Centos 7 network)',
                      inline: 'systemctl restart network'

  # ----------------------------------------------------------------------------
  # SSH configuration (apply to all VMs)
  # ----------------------------------------------------------------------------

  # Use multiple private keys
  # Set up configuration to use two or more private keys.
  # The key that the box has at first is `~/.vagrant.d/insecure_private_key` so
  # you should append this default key. Vagrant tries using private keys in
  # order, so place your major key first.
  # TODO: test security with insecure_private_key
  config.ssh.private_key_path = [
    '.ssh/id_rsa',
    "#{Dir.home}/.vagrant.d/insecure_private_key" # to provision the first time
  ]

  # Do not generate a key
  # Vagrant generates a random key and insert to box, and this random key is not
  # in the private keys to use (set up above). So you cannot login. So we modify
  # setting to not insert randomly generated key in the box.
  config.ssh.insert_key = false

  # Copy the public key into the box.
  config.vm.provision 'file',
                      source: '.ssh/id_rsa.pub',
                      destination: '~/.ssh/id_rsa.pub'

  # Copy the private key into the box.
  config.vm.provision 'file',
                      source: '.ssh/id_rsa',
                      destination: '~/.ssh/id_rsa'

  # Fix permissions.
  config.vm.provision 'shell',
                      privileged: false,
                      name: 'Fix permissions',
                      inline: <<-SHELL
                        chmod 600 /home/vagrant/.ssh/id_rsa
                        chmod 644 /home/vagrant/.ssh/id_rsa.pub
                      SHELL

  # Automatically add *.dev hosts to known_hosts by disabling the StrictHostKeyChecking
  config.vm.provision 'shell',
                      name: 'Configure StrictHostKeyChecking',
                      inline: <<-SHELL
                        sudo printf '%s\n' 'Host *.dev' '  StrictHostKeyChecking no' > /home/vagrant/.ssh/config
                        sudo chown vagrant:vagrant /home/vagrant/.ssh/config
                        sudo chmod 400 /home/vagrant/.ssh/config
                      SHELL

  # Prevent access with plaintext password
  # The box allows login with plaintext password so attacker can login with
  # default username and password vagrant / vagrant.
  config.vm.provision 'shell',
                      name: 'Prevent access with plaintext password',
                      inline: <<-SHELL
                        sudo sed -i -e "\\#PasswordAuthentication yes# s#PasswordAuthentication yes#PasswordAuthentication no#g" /etc/ssh/sshd_config
                        sudo systemctl restart sshd
                      SHELL

  # Append the `personal` and `project` public SSH key to the clients' authorized_keys.
  config.vm.provision 'shell',
                      privileged: false,
                      name: 'Apply host\'s id_rsa.pub to vagrant authorized_keys',
                      inline: <<-SHELL
                        # Print the key | add to `authorized_keys`, sort the files, remove duplicates  | write to `authorized_keys_tmp`
                        echo '#{ssh_public_key_project}' 2>&1 | sort -u - /home/vagrant/.ssh/authorized_keys > /home/vagrant/.ssh/authorized_keys_tmp
                        mv /home/vagrant/.ssh/authorized_keys_tmp /home/vagrant/.ssh/authorized_keys

                        # Do the same for the other key
                        echo '#{ssh_public_key_personal}' 2>&1 | sort -u - /home/vagrant/.ssh/authorized_keys > /home/vagrant/.ssh/authorized_keys_tmp
                        mv /home/vagrant/.ssh/authorized_keys_tmp /home/vagrant/.ssh/authorized_keys

                        # Remove Vagrant insecure public key
                        sed -i '/vagrant insecure public key/d' /home/vagrant/.ssh/authorized_keys

                        # Fix permissions
                        chmod 644 /home/vagrant/.ssh/authorized_keys
                      SHELL

  # ----------------------------------------------------------------------------
  # Vagrant Host Manager (apply to all VMs)
  # ----------------------------------------------------------------------------

  # User the Vagrant plugin Hostmanager to add on host's and on clients' hosts
  # file the VMs using the hostname and the private IP.
  config.hostmanager.enabled = true
  config.hostmanager.manage_host = true
  config.hostmanager.manage_guest = true
  config.hostmanager.ignore_private_ip = false
  config.hostmanager.include_offline = true

  # ----------------------------------------------------------------------------
  # VM creation (apply to all VMs)
  # ----------------------------------------------------------------------------

  # For every VM in the vm_array create an instance.
  vm_array.each do |vm_current|
    # As Vagrant VM name use the URL without the suffix.
    config.vm.define vm_current[0].chomp('.dev') do |node|
      # The hostname the machine should have.
      node.vm.hostname = vm_current[0]
      # Create a private network, which allows host-only access to the machine
      # using a specific IP.
      node.vm.network 'private_network', ip: vm_current[1]

      # Print message after VM creation.
      login_string = []
      if File.exist?(my_public_ssh_key)
        login_string[0] = "ssh vagrant@#{vm_current[0]}"
        login_string[1] = "ssh vagrant@#{vm_current[1]}"
        login_string[2] = "vagrant ssh #{vm_current[0].chomp('.dev')}"
        login_string[3] = my_public_ssh_key.to_s
      else
        login_string[0] = "ssh -i #{Dir.pwd}/.ssh/id_rsa vagrant@#{vm_current[0]}"
        login_string[1] = "ssh -i #{Dir.pwd}/.ssh/id_rsa vagrant@#{vm_current[1]}"
        login_string[2] = "vagrant ssh #{vm_current[0].chomp('.dev')}"
        login_string[3] = ''
      end

      node.vm.post_up_message = "
      -------------------------
      `#{vm_current[0].chomp('.dev')}` server is ready
      IP:       #{vm_current[0]}
      Hostname: #{vm_current[1]}
      Login:    #{login_string[0]}
                #{login_string[1]}
                #{login_string[2]}
      SSH keys: #{Dir.pwd}/.ssh/id_rsa
                #{login_string[3]}
      -------------------------"
    end
  end

  # ----------------------------------------------------------------------------
  # Extra configuration per VM
  # ----------------------------------------------------------------------------

  # If you want extra configuration for a specific VM add:
  #
  # config.vm.define VM_NAME do |node|
  #   node.vm.PROPERTY = VARIABLE
  # end

  # Redefine VM ansible and apply extra provision.
  config.vm.define 'ansible' do |custom|
    config.vm.provision 'file',
                        source: 'hosts_multivm',
                        destination: '~/hosts_multivm'

    custom.vm.provision 'shell',
                        name: 'Shell provisioning',
                        inline: <<-SHELL
                          echo "\n\nAdd EPEL repository public key\n------------------------------"
                          rpm --import https://getfedora.org/static/352C64E5.txt
                          rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

                          echo "\n\nInstall EPEL repository\n------------------------------"
                          if ! rpm -qa | grep ^epel-release; then
                            yum install -y epel-release
                          fi

                          echo "\n\nInstall Ansible\n------------------------------"
                          if ! rpm -qa | grep ^ansible; then
                            yum install -y ansible
                          fi

                          echo "\n\nInstall git\n------------------------------"
                          if ! rpm -qa | grep ^git; then
                            yum install -y git
                          fi

                          echo "\n\nDownload the Ansible playbook\n------------------------------"
                          if [ ! -d "/home/vagrant/ansible-playbook-aegir-centos" ]; then
                            git clone --recursive https://github.com/tovletoglou/ansible-playbook-aegir-centos.git /home/vagrant/ansible-playbook-aegir-centos
                          fi

                          echo "\n\nChange playbook owner\n------------------------------"
                          chown -R vagrant:vagrant /home/vagrant/ansible-playbook-aegir-centos

                          echo "\n\nRun playbook_ansible\n------------------------------"
                          runuser -l vagrant -c 'ansible-playbook -i /home/vagrant/hosts_multivm /home/vagrant/ansible-playbook-aegir-centos/playbook_ansible.yml'

                          echo "\n\nRun playbook_vagrant\n------------------------------"
                          runuser -l vagrant -c 'ansible-playbook -i /home/vagrant/hosts_multivm /home/vagrant/ansible-playbook-aegir-centos/playbook_vagrant.yml'

                          echo "\n\nRun playbook_hostmaster\n------------------------------"
                          runuser -l vagrant -c 'ansible-playbook -i /home/vagrant/hosts_multivm /home/vagrant/ansible-playbook-aegir-centos/playbook_hostmaster.yml'

                        SHELL
  end
end
