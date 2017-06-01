$consul_install = <<CONSUL_INST
CONSUL_VERSION=0.8.3
CONSUL_ZIP=/vagrant/bin/consul_${CONSUL_VERSION}_linux_amd64.zip
if [ ! -f $CONSUL_ZIP ]; then
  mkdir -p /vagrant/bin
  wget https://releases.hashicorp.com/consul/${CONSUL_VERSION}/consul_${CONSUL_VERSION}_linux_amd64.zip --quiet -O $CONSUL_ZIP
fi
unzip -q $CONSUL_ZIP
sudo chmod +x consul
sudo mv consul /usr/local/bin/consul
sudo mkdir -pm 0600 /etc/systemd/system/consul.d

# setup consul directories
sudo mkdir -pm 0600 /opt/consul
sudo mkdir -p /opt/consul/data
CONSUL_INST

$consul_config = <<CONSUL_CONFIG
sudo cat << EOF > /etc/systemd/system/consul.d/consul.json
{
  "datacenter": "vagrant",
  "node_name": "$(hostname -f)",
  "data_dir": "/opt/consul/data",
  "ui": true,
  "client_addr": "0.0.0.0",
  "bind_addr": "0.0.0.0",
  "advertise_addr": "$(/usr/sbin/ifconfig enp0s8 | grep 'inet ' | awk '{print $2}')",
  "leave_on_terminate": false,
  "skip_leave_on_interrupt": true,
  "server": true,
  "bootstrap_expect": 3
}
EOF
CONSUL_CONFIG

$consul_svc = <<CONSUL_SVC
sudo cat << EOF > /etc/systemd/system/consul.service
[Unit]
Description=consul agent
Requires=network-online.target
After=network-online.target

[Service]
EnvironmentFile=-/etc/default/consul
Restart=on-failure
ExecStart=/usr/local/bin/consul agent $CONSUL_FLAGS -config-dir=/etc/systemd/system/consul.d
ExecReload=/bin/kill -HUP $MAINPID
KillSignal=SIGINT

[Install]
WantedBy=multi-user.target
EOF
CONSUL_SVC

$consul_join = <<CONSUL_JOIN
if [ "$HOSTNAME" != "vault1" ]; then
/usr/local/bin/consul join $(cat /vagrant/consul_join)
fi
CONSUL_JOIN

$vault_install = <<VAULT_INST
VAULT=0.7.2
VAULT_ZIP=/vagrant/bin/vault_${VAULT}_linux_amd64.zip

if [ ! -f $VAULT_ZIP ]; then
  mkdir -p /vagrant/bin
  wget https://releases.hashicorp.com/vault/${VAULT}/vault_${VAULT}_linux_amd64.zip --quiet -O $VAULT_ZIP
fi
cd /tmp
unzip -q $VAULT_ZIP >/dev/null
sudo chmod +x vault
sudo mv vault /usr/local/bin
sudo chmod 0755 /usr/local/bin/vault
sudo chown root:root /usr/local/bin/vault
#sudo setcap cap_ipc_lock=+ep /usr/local/bin/vault
sudo mkdir -pm 0755 /etc/vault

sudo /usr/sbin/groupadd --force --system vault

if ! getent passwd vault >/dev/null ; then
        sudo /usr/sbin/adduser \
          --system \
          --gid vault \
          --no-create-home \
          --comment "HashiCorp Vault User" \
          --shell /bin/false \
          vault  >/dev/null
fi
VAULT_INST

$vault_env = <<VAULT_ENV
sudo cat << EOF > /etc/profile.d/vault.sh
export VAULT_ADDR="http://127.0.0.1:8200"
EOF
VAULT_ENV

$vault_config = <<VAULT_CONFIG
sudo cat << EOF > /etc/vault/config.hcl
backend "consul" {
  address = "127.0.0.1:8500"
  path = "vault"
}
listener "tcp" {
  address = "0.0.0.0:8200"
  tls_disable = 1
}
EOF

VAULT_CONFIG

$vault_svc = <<VAULT_SVC
sudo cat << EOF > /etc/systemd/system/vault.service
[Unit]
Description=vault agent
Requires=network-online.target
After=network-online.target

[Service]
Restart=on-failure
User=vault
Group=vault
PermissionsStartOnly=true
ExecStartPre=/sbin/setcap 'cap_ipc_lock=+ep' /usr/local/bin/vault
ExecStart=/usr/local/bin/vault server -config /etc/vault
ExecReload=/bin/kill -HUP $MAINPID
KillSignal=SIGINT

[Install]
WantedBy=multi-user.target
EOF
sudo chmod 0644 /etc/systemd/system/vault.service
VAULT_SVC



Vagrant.configure("2") do |config|
  config.vm.network :forwarded_port, guest: 8200, host: 8200, auto_correct: true
  config.vm.network :forwarded_port, guest: 8500, host: 8500, auto_correct: true
  config.vm.provider "virtualbox" do |vb|
    vb.customize ["modifyvm", :id, "--memory", "512"]
    vb.customize ["modifyvm", :id, "--cpus", "1"]
    vb.customize ["modifyvm", :id, "--cpuexecutioncap", "50"]
    vb.customize ["modifyvm", :id, "--chipset", "ich9"]
    vb.customize ["modifyvm", :id, "--ioapic", "on"]
  end
  config.vm.define "vault1" do |vault1|
    vault1.vm.box = "bento/centos-7.3"
    vault1.vm.hostname = "vault1"
    vault1.vm.network "private_network", type: "dhcp"
    vault1.vm.provision "shell", inline: "sudo /usr/sbin/ifconfig enp0s8 | grep 'inet ' | awk '{print $2}' > /vagrant/consul_join"
  end

  config.vm.define "vault2" do |vault2|
    vault2.vm.box = "bento/centos-7.3"
    vault2.vm.hostname = "vault2"
    vault2.vm.network "private_network", type: "dhcp"
  end

  config.vm.define "vault3" do |vault3|
    vault3.vm.box = "bento/centos-7.3"
    vault3.vm.hostname = "vault3"
    vault3.vm.network "private_network", type: "dhcp"
  end

  config.vm.provision "shell", inline: "sudo yum install -q -y wget vim-enhanced unzip ca-certificates"
  config.vm.provision "shell", inline: "sudo curl -s -L https://github.com/stedolan/jq/releases/download/jq-1.5/jq-linux64 > /usr/local/bin/jq"
  config.vm.provision "shell", inline: "sudo chmod +x /usr/local/bin/jq"
  config.vm.provision "shell", inline: $consul_install
  config.vm.provision "shell", inline: $consul_config
  config.vm.provision "shell", inline: $consul_svc
  config.vm.provision "shell", inline: $vault_install
  config.vm.provision "shell", inline: $vault_env
  config.vm.provision "shell", inline: $vault_config
  config.vm.provision "shell", inline: $vault_svc
  config.vm.provision "shell", inline: "sudo systemctl enable consul.service"
  config.vm.provision "shell", inline: "sudo systemctl start consul"
  config.vm.provision "shell", inline: $consul_join
  config.vm.provision "shell", inline: "sudo systemctl enable vault.service"
  config.vm.provision "shell", inline: "sudo systemctl start vault"
  config.vm.provision "shell", inline: "sudo cp /vagrant/vault_init_and_unseal.sh /root/"
  config.vm.provision "shell", inline: "sudo cp /vagrant/vault_postgres.sh /root/"
  config.vm.provision "shell", inline: "sudo chmod +x /root/*.sh"

end
