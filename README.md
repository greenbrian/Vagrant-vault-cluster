# vagrant-vault-cluster

This repo contains scripts and Vagrantfiles to spin up either a single node with Vault + Consul or a 3 node setup with Vault + Consul

## Single node instructions
Within this repository diretory as working directory execute the following:

```
SERVER_COUNT="1" vagrant up vault1
```

Then you can execute the following to login to the virtual machine

```
vagrant ssh vault1
```

## Three node instructions
Within this repository diretory as working directory execute the following:

```
vagrant up
```

Then you can execute the following to login to the virtual machine

```
vagrant ssh vault1
```

---



## Using this guide as a reference for installation on servers?

To install, perform the following steps in order. 

1. Copy and execute `scripts\base.sh` 
1. Copy and execute `scripts\install-consul.sh`
1. Copy and execute `scripts\install-vault.sh` 
1. If using Enterprise binaries replace the Vault and Consul binaries with their enterprise equivalents. 
1. Copy and execute `scripts\install-systemd-scripts.sh`
1. Copy `scripts\install-configs.sh` to server. 
	1. Edit the relevant Consul section with a list of the cluster IP addresses in `retry_join`[documentation link](https://www.consul.io/docs/agent/options.html#retry_join)
 	1. Edit the relevant Consul section with the `advertise_addr` specific to each server. Consul [documentation link](https://www.consul.io/docs/agent/options.html#advertise_addr)
1. Execute `install-configs.sh 1` or `install-configs.sh 3` for appropriate number of Consul servers. 


Last: To start services, execute the following on each node:

```
sudo systemctl enable consul.service
sudo systemctl start consul
sudo systemctl enable vault.service
sudo systemctl start vault
```
