# Vault Deployment

This deployment guide covers the steps required to install and configure a single HashiCorp Vault cluster.

## Step 1: Download Vault

Precompiled Vault binaries are available for download at https://releases.hashicorp.com/vault/ 

export VAULT_URL="https://releases.hashicorp.com/vault" VAULT_VERSION="1.5.0"

Then use curl to download the package and SHA256 summary files.


curl \
    --silent \
    --remote-name \
   "${VAULT_URL}/${VAULT_VERSION}/vault_${VAULT_VERSION}_linux_amd64.zip"

Use curl to download the package and SHA256 summary files.

curl \
    --silent \
    --remote-name \
    "${VAULT_URL}/${VAULT_VERSION}/vault_${VAULT_VERSION}_SHA256SUMS"

curl \
    --silent \
    --remote-name \
    "${VAULT_URL}/${VAULT_VERSION}/vault_${VAULT_VERSION}_SHA256SUMS.sig"

You should now have the 3 files present locally:

$ ls -1

## Step 2: Install Vault

Unzip the downloaded package and move the vault binary to /usr/local/bin/.

unzip vault_${VAULT_VERSION}_linux_amd64.zip

Set the owner of the Vault binary.

sudo chown root:root vault

Check vault is available on the system path.

sudo mv vault /usr/local/bin/

You can verify the Vault version.

vault --version

The vault command features opt-in autocompletion for flags, subcommands, and arguments (where supported).


vault -autocomplete-install

Enable autocompletion.

complete -C /usr/local/bin/vault vault

Give Vault the ability to use the mlock syscall without running the process as root. The mlock syscall prevents memory from being swapped to disk.

sudo setcap cap_ipc_lock=+ep /usr/local/bin/vault

Create a unique, non-privileged system user to run Vault.

sudo useradd --system --home /etc/vault.d --shell /bin/false vault

## Step 3: Configure systemd

Create a Vault service file at /etc/systemd/system/vault.service.

sudo touch /etc/systemd/system/vault.service

Add the below configuration to the Vault service file:

[Unit]
Description="HashiCorp Vault - A tool for managing secrets"
Documentation=https://www.vaultproject.io/docs/
Requires=network-online.target
After=network-online.target
ConditionFileNotEmpty=/etc/vault.d/vault.hcl
StartLimitIntervalSec=60
StartLimitBurst=3

[Service]
User=vault
Group=vault
ProtectSystem=full
ProtectHome=read-only
PrivateTmp=yes
PrivateDevices=yes
SecureBits=keep-caps
AmbientCapabilities=CAP_IPC_LOCK
Capabilities=CAP_IPC_LOCK+ep
CapabilityBoundingSet=CAP_SYSLOG CAP_IPC_LOCK
NoNewPrivileges=yes
ExecStart=/usr/local/bin/vault server -config=/etc/vault.d/vault.hcl
ExecReload=/bin/kill --signal HUP $MAINPID
KillMode=process
KillSignal=SIGINT
Restart=on-failure
RestartSec=5
TimeoutStopSec=30
StartLimitInterval=60
StartLimitIntervalSec=60
StartLimitBurst=3
LimitNOFILE=65536
LimitMEMLOCK=infinity

[Install]
WantedBy=multi-user.target


