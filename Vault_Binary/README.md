# Vault Cluster Setup Documentation Using Vagrant(Without KMS)

This documentation provides a step-by-step guide to setting up a HashiCorp Vault Cluster using Vagrant and Libvirt. The cluster consists of three Vault nodes configured to run in a Raft-based storage mode for high availability.

## Prerequisites

* Vagrant installed.

* Libvirt provider installed (for example, using vagrant-libvirt plugin).

* Ubuntu 22.04 (generic/ubuntu2204) Vagrant box available.

* Access to terminal/command-line with sudo privileges.

## Step 1: Vagrant Configuration

```bash
nano Vagrantfile

Vagrant.configure("2") do |config|
  config.vm.box = "generic/ubuntu2204"

  # Vault1 VM
  config.vm.define "vault1" do |vault1|
    vault1.vm.hostname = "vault1"
    vault1.vm.network "private_network", ip: "192.168.56.127"  # Unique IP for vault1
    vault1.vm.provider :libvirt do |lv|
      lv.memory = 4096  # 4 GB of memory for vault1 VM
      lv.cpus = 2       # 2 CPUs for vault1 VM
    end
  end

  # Vault2 VM
  config.vm.define "vault2" do |vault2|
    vault2.vm.hostname = "vault2"
    vault2.vm.network "private_network", ip: "192.168.56.128"  # Unique IP for vault2
    vault2.vm.provider :libvirt do |lv|
      lv.memory = 4096  # 4 GB of memory for vault2 VM
      lv.cpus = 2       # 2 CPUs for vault2 VM
    end
  end

  # Vault3 VM
  config.vm.define "vault3" do |vault3|
    vault3.vm.hostname = "vault3"
    vault3.vm.network "private_network", ip: "192.168.56.129"  # Unique IP for vault3
    vault3.vm.provider :libvirt do |lv|
      lv.memory = 4096  # 4 GB of memory for vault3 VM
      lv.cpus = 2       # 2 CPUs for vault3 VM
    end
  end
end



# vagrant up
```


## Step 2: SSH into Each VM

```bash
# Terminal 1: SSH into vault1
vagrant ssh vault1

# Terminal 2: SSH into vault2
vagrant ssh vault2

# Terminal 3: SSH into vault3
vagrant ssh vault3

```


## Step 3: Install Vault
For each VM, switch to root and create an installation script to install Vault.
```bash

# 1. Switch to Root User
sudo su

# 2. Create install_vault.sh Script
nano install_vault.sh

#!/bin/bash
apt-get install -y unzip

USER="vault"
COMMENT="Hashicorp vault user"
GROUP="vault"
HOME="/srv/vault"

# Add Vault user and group
sudo addgroup --system ${GROUP} >/dev/null
sudo adduser \
--system \
--disabled-login \
--ingroup ${GROUP} \
--home ${HOME} \
--no-create-home \
--gecos "${COMMENT}" \
--shell /bin/false \
${USER} >/dev/null

# Download Vault
cd /opt/ && sudo curl -o vault.zip https://releases.hashicorp.com/vault/1.13.1/vault_1.13.1_linux_amd64.zip
sudo unzip vault.zip
sudo mv vault /usr/local/bin/
mkdir -pm 0755 /etc/vault.d
sudo chown -R vault:vault /etc/vault.d
sudo mkdir /vault-data
sudo chown -R vault:vault /vault-data
mkdir -pm 0755 /opt/vault
chown vault:vault /opt/vault
chown vault:vault /usr/local/bin/vault

# Create Vault systemd service
cat << EOF > /lib/systemd/system/vault.service
[Unit]
Description=Vault Agent
Requires=network-online.target
After=network-online.target
[Service]
Restart=on-failure
PermissionsStartOnly=true
ExecStartPre=/sbin/setcap 'cap_ipc_lock=+ep' /usr/local/bin/vault
ExecStart=/usr/local/bin/vault server -config /etc/vault.d
ExecReload=/bin/kill -HUP $MAINPID
KillSignal=SIGTERM
User=vault
Group=vault
[Install]
WantedBy=multi-user.target
EOF

# 3. Make the Script Executable

chmod +x install_vault.sh

# 4. Run the Script
./install_vault.sh


```


## Step 4: Configure Vault on Each Node

Each node (vault1, vault2, vault3) needs a custom vault.hcl configuration file.
Create the directory for Vault’s configuration.

```bash
sudo mkdir -pm 0755 /etc/vault.d

#Terminal 1
cat << EOF > /etc/vault.d/vault.hcl
storage "raft" {
  path    = "/opt/vault"
  node_id = "vault-node-1"

  retry_join {
    leader_api_addr = "http://192.168.56.128:8200"
  }
  retry_join {
    leader_api_addr = "http://192.168.56.129:8200"
  }
}

listener "tcp" {
  address         = "0.0.0.0:8200"
  cluster_address = "0.0.0.0:8201"
  tls_disable     = 1
}

cluster_addr = "http://192.168.56.127:8201"
api_addr     = "http://192.168.56.127:8200"
ui = true
EOF

# Terminal 2
cat << EOF > /etc/vault.d/vault.hcl
storage "raft" {
  path    = "/opt/vault"
  node_id = "vault-node-2"

  retry_join {
    leader_api_addr = "http://192.168.56.127:8200"
  }
  retry_join {
    leader_api_addr = "http://192.168.56.129:8200"
  }
}

listener "tcp" {
  address         = "0.0.0.0:8200"
  cluster_address = "0.0.0.0:8201"
  tls_disable     = 1
}

cluster_addr = "http://192.168.56.128:8201"
api_addr     = "http://192.168.56.128:8200"
ui = true
EOF


# Terminal 3
cat << EOF > /etc/vault.d/vault.hcl
storage "raft" {
  path    = "/opt/vault"
  node_id = "vault-node-3"

  retry_join {
    leader_api_addr = "http://192.168.56.127:8200"
  }
  retry_join {
    leader_api_addr = "http://192.168.56.128:8200"
  }
}

listener "tcp" {
  address         = "0.0.0.0:8200"
  cluster_address = "0.0.0.0:8201"
  tls_disable     = 1
}

cluster_addr = "http://192.168.56.129:8201"
api_addr     = "http://192.168.56.129:8200"
ui = true
EOF


```


## Step 5: Enable and Start Vault
For each terminal, run the following commands to enable and start Vault:

```bash
sudo chmod 0664 /lib/systemd/system/vault.service
systemctl daemon-reload
sudo chmod -R 0644 /etc/vault.d/*
chmod 0755 /usr/local/bin/vault
cat << EOF > /etc/profile.d/vault.sh
export VAULT_ADDR=http://127.0.0.1:8200
export VAULT_SKIP_VERIFY=true
EOF
systemctl enable vault
systemctl start vault
export VAULT_ADDR=http://127.0.0.1:8200

```

## Step 6: Initialize and Unseal Vault

Initialize Vault (on Vault1)

Only run the following commands on the Vault master node (Vault1):

```bash
vault operator init

vault operator unseal <key1>
vault operator unseal <key2>
vault operator unseal <key3>


vault login <root key>

vault operator raft list-peers

```
## Step 7: Unseal  Nodes-2 Uaing Master Node Key:

```bash

Terminal-2
vault operator unseal <key1>
vault operator unseal <key2>
vault operator unseal <key3>


```

## Step 8: Unseal  Nodes-3 Uaing Master Node Key:
```bash
Terminal-3
vault operator unseal <key1>
vault operator unseal <key2>
vault operator unseal <key3>

```

## Step 9 : Verify Vault Cluster Status

```bash
vault login <root key>
vault operator raft list-peers
```
## Output

```bash
Node      Address                    State       Voter
----      -------                    -----       -----
vault-node-1    192.168.56.127:8201    leader      true
vault-node-2    192.168.56.128:8201    follower    true
vault-node-3    192.168.56.129:8201    follower    true
```


## Step 10: Fault Tolerance Test

This step demonstrates Vault's high availability and fault tolerance capabilities by testing failover when the leader node goes down.
1. Create a New Secret Engine and Secret

```bash
# Enable a new KV secret engine
vault secrets enable -path=test2 -version=2 kv

# Store a secret in the new engine
vault kv put test2/test2 Mongo=mypass

# Verify the stored secret
vault kv get test2/test2
```
Expected Output:

```bash
== Secret Path ==
test2/data/test2

======= Metadata =======
Key                Value
---                -----
created_time       2025-11-26T12:29:34.634451158Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1

==== Data ====
Key      Value
---      -----
Mongo    mypass
```
2. Stop the Leader Node
Simulate a leader failure by stopping Vault on the leader node:

```bash
# On vault1 (leader node)
sudo systemctl stop vault

# Verify Vault is stopped
sudo systemctl status vault
```
Expected Status:

```bash
○ vault.service - "HashiCorp Vault - A tool for managing secrets"
     Loaded: loaded (/lib/systemd/system/vault.service; enabled; vendor preset: enabled)
     Active: inactive (dead) since Wed 2025-11-26 12:30:54 UTC; 8s ago
```

3. Test Secret Access on Another Node
Access the same secret from a different node (vault2) to verify failover:
```bash
# On vault2/vault3 (follower node)
# First login with the same root token (if not already logged in)
vault login  <root Pass>

# Then access the secret
vault kv get test2/test2
```
Expected Output:

```bash
== Secret Path ==
test2/data/test2

======= Metadata =======
Key                Value
---                -----
created_time       2025-11-26T12:29:34.634451158Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1

==== Data ====
Key      Value
---      -----
Mongo    mypass
```

## ⚠️ Important Cluster Availability Notice

Critical Quorum Requirements:

* 3-Node Cluster: Can tolerate 1 node failure without service disruption

* 5-Node Cluster: Can tolerate 2 node failures without service disruption

## What Happens During Multiple Node Failures:

### Scenario 1: Two Nodes Down Simultaneously

* If vault1 AND vault2 are down simultaneously:
* Cluster loses quorum (2 out of 3 nodes unavailable)
* Vault becomes READ-ONLY or completely UNAVAILABLE
* No new writes can be performed until quorum is restored

### Scenario 2: Node Restart + Another Node Failure

* If vault1 is restarting (unsealed) AND vault2 goes down:
* Cluster cannot maintain quorum
* Operations may hang or fail
* Automatic leader election cannot complete

## Recovery from Quorum Loss:

### To recover when quorum is lost:
 1. Bring at least 2 nodes back online
 2. Ensure all nodes are unsealed
 3. Wait for raft consensus to be reestablished
 4. Verify cluster health

```BASH
vault operator raft list-peers
vault status
```