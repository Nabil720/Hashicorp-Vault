# Vault Cluster Setup Documentation Using Vagrant

This documentation provides a step-by-step guide to setting up a HashiCorp Vault Cluster using Vagrant and Libvirt. The cluster consists of three Vault nodes configured to run in a Raft-based storage mode for high availability.

## Prerequisites

* Vagrant installed.

* Libvirt provider installed (for example, using vagrant-libvirt plugin).

* Ubuntu 22.04 (generic/ubuntu2204) Vagrant box available.

* Access to terminal/command-line with sudo privileges.

## Step 1: Vagrant Configuration

Create Vgrantfile:
```bash
nano Vagrantfile
```

```bash
Vagrant.configure("2") do |config|
  config.vm.box = "generic/ubuntu2204"

  # Vault1 VM
  config.vm.define "vault1" do |vault1|
    vault1.vm.hostname = "vault1"
    vault1.vm.network "private_network", ip: "192.168.56.130"  # Unique IP for vault1
    vault1.vm.provider :libvirt do |lv|
      lv.memory = 4096  # 4 GB of memory for vault1 VM
      lv.cpus = 2       # 2 CPUs for vault1 VM
    end
  end

  # Vault2 VM
  config.vm.define "vault2" do |vault2|
    vault2.vm.hostname = "vault2"
    vault2.vm.network "private_network", ip: "192.168.56.131"  # Unique IP for vault2
    vault2.vm.provider :libvirt do |lv|
      lv.memory = 4096  # 4 GB of memory for vault2 VM
      lv.cpus = 2       # 2 CPUs for vault2 VM
    end
  end

  # Vault3 VM
  config.vm.define "vault3" do |vault3|
    vault3.vm.hostname = "vault3"
    vault3.vm.network "private_network", ip: "192.168.56.132"  # Unique IP for vault3
    vault3.vm.provider :libvirt do |lv|
      lv.memory = 4096  # 4 GB of memory for vault3 VM
      lv.cpus = 2       # 2 CPUs for vault3 VM
    end
  end
end
```
Create VM Using Vagrantfile:

```bash
vagrant up
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

### On each Vault node
```bash
sudo apt update
sudo apt install -y wget gpg lsb-release unzip curl zip openssl

wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt update

```

* On each Vault node


```bash
apt list -a vault

sudo apt install vault -y
vault version


```



## AWS  Part
### 1. Create KMS Key For Vault Unsealing Process
-   **Key Type:** `Symmetric`
-   **Key Usage:** `Encrypt` and `Decrypt`
-   Attach the **default key policy** (or customize for your IAM users).
-   **Important:** Note down the **KMS Key ID**

 ### 2. Create IAM Policy & Role To Use This KMS Key

- Policy should be 
```bash
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VaultKMSUnseal",
            "Effect": "Allow",
            "Action": [
                "kms:Encrypt",
                "kms:Decrypt",
                "kms:DescribeKey"
            ],
            "Resource": "arn:aws:kms:ap-sou******-4f1c-b04a-7ad91c3cfac3"
        }
    ]
}
```

### Step 3: Create IAM User and Attach Policy

* Go to IAM → Users → Create user

* User name example:
      `vault-kms-user`

* Attach the policy created in Step 2

* Complete the user creation

* Save the following credentials securely:
  * AWS Access Key ID
  * AWS Secret Access Key


### Step 4: Create Vault Environment File

Create the environment file:

```bash
sudo mkdir -p /etc/vault.d 
sudo vi /etc/vault.d/vault.env


AWS_ACCESS_KEY_ID=AKIAxxxxxxxxxxxx
AWS_SECRET_ACCESS_KEY=xxxxxxxxxxxxxxxxxxxxxxxxxxxx 
AWS_DEFAULT_REGION=ap-south-1
```
Set correct permissions:
```bash
sudo chown vault:vault /etc/vault.d/vault.env 
sudo chmod 600 /etc/vault.d/vault.env
```


### Step 5: Update Vault Systemd Service File
Edit the Vault systemd service file:
```bash
sudo vi /usr/lib/systemd/system/vault.service


[Service] 
EnvironmentFile=/etc/vault.d/vault.env
```
Reload systemd and restart Vault:
```bash
sudo systemctl daemon-reload 
sudo systemctl restart vault 
sudo systemctl status vault
```

## Step 4: Configure Vault on Each Node

Each node (vault1, vault2, vault3) needs a custom vault.hcl configuration file.
Create the directory for Vault’s configuration.

Get Sudo permission :

```bash
sudo su
```
### Terminal-1 vault1

```bash
cat << EOF > /etc/vault.d/vault.hcl
disable_mlock = true
storage "raft" {
  path    = "/opt/vault"
  node_id = "vault-node-1"

  retry_join {
    leader_api_addr = "http://192.168.56.131:8200"   # IP of vault2   
  }
  retry_join {
    leader_api_addr = "http://192.168.56.132:8200"   # IP of vault3
  }
}

listener "tcp" {
  address         = "0.0.0.0:8200"
  cluster_address = "0.0.0.0:8201"
  tls_disable     = 1
}

seal "awskms" {
  region     = "ap-south-1"
  kms_key_id = "f79****-7ad91c3cfac3"
}

cluster_addr = "http://192.168.56.130:8201"   # IP of vault1
api_addr     = "http://192.168.56.130:8200"   # IP of vault1
ui = true
EOF
```


### Terminal-2 vault2

```bash
cat << EOF > /etc/vault.d/vault.hcl
disable_mlock = true
storage "raft" {
  path    = "/opt/vault"
  node_id = "vault-node-2"

  retry_join {
    leader_api_addr = "http://192.168.56.130:8200"   # IP of vault1
  }
  retry_join {
    leader_api_addr = "http://192.168.56.132:8200"   # IP of vault3
  }
}

listener "tcp" {
  address         = "0.0.0.0:8200"
  cluster_address = "0.0.0.0:8201"
  tls_disable     = 1
}

seal "awskms" {
  region     = "ap-south-1"
  kms_key_id = "f79****-7ad91c3cfac3"
}


cluster_addr = "http://192.168.56.131:8201"   # IP of vault2
api_addr     = "http://192.168.56.131:8200"   # IP of vault2
ui = true
EOF
```



### Terminal-3 vault3

```bash
cat << EOF > /etc/vault.d/vault.hcl
disable_mlock = true
storage "raft" {
  path    = "/opt/vault"
  node_id = "vault-node-3"

  retry_join {
    leader_api_addr = "http://192.168.56.130:8200"   # IP of vault1
  }
  retry_join {
    leader_api_addr = "http://192.168.56.131:8200"   # IP of vault2
  }
}

listener "tcp" {
  address         = "0.0.0.0:8200"
  cluster_address = "0.0.0.0:8201"
  tls_disable     = 1
}

seal "awskms" {
  region     = "ap-south-1"
  kms_key_id = "f79****-7ad91c3cfac3"
}


cluster_addr = "http://192.168.56.132:8201"   # IP of vault3
api_addr     = "http://192.168.56.132:8200"   # IP of vault3
ui = true
EOF


```


## Step 5: Enable and Start Vault
For each Vault node, run the following commands to enable and start Vault:

```bash
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
## Output

```bash
Node      Address                    State       Voter
----      -------                    -----       -----
vault-node-1    192.168.56.130:8201    leader      true
```

## Step 7: Unseal  Nodes-2 Uaing Master Node Key:

To add other Vault nodes to the cluster (Vault2, Vault3), So I need to unseal each of them using the unseal keys. This is important, because without unsealing, Vault will not join the cluster.

```bash

Terminal-2
vault operator unseal <key1>
vault operator unseal <key2>
vault operator unseal <key3>


```

## Step 8: Unseal  Nodes-3 Uaing Master Node Key:

To add other Vault nodes to the cluster (Vault2, Vault3), So I need to unseal each of them using the unseal keys. This is important, because without unsealing, Vault will not join the cluster.

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
vault-node-1    192.168.56.130:8201    leader      true
vault-node-2    192.168.56.131:8201    follower    true
vault-node-3    192.168.56.132:8201    follower    true
```


## ⚠️ Fault Tolerance Test

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


```bash
vault login <root key>
vault operator raft list-peers
```
## Output

```bash
Node      Address                    State       Voter
----      -------                    -----       -----
vault-node-1    192.168.56.130:8201    leader      true
vault-node-2    192.168.56.131:8201    follower    true
vault-node-3    192.168.56.132:8201    follower    true
```

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