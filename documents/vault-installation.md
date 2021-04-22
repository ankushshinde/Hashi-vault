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

## Step 4: Configure Consul

When using Consul as the storage backend for Vault, we recommend using Consul's ACL system to restrict access to the path where Vault stores data. This access restriction is an added security measure in addition to the encryption Vault uses to protect data written to the storage backend.

On a host running a Consul agent:

- Export the CONSUL_HTTP_ADDR environment variable:

export CONSUL_HTTP_ADDR="127.0.0.1:8500"

- Update Consul configuration in /etc/consul.d/consul.hcl on all Consul Servers to use ACLs and choose a default policy of allow (like a soft limit) or deny (like a hard limit).

acl_datacenter =  "dc1"
acl_default_policy =  "deny"
acl_down_policy =  "extend-cache"

- Restart Consul.

sudo systemctl restart consul

- Generate the Consul ACL Management token.

curl --request PUT https://"${CONSUL_HTTP_ADDR}"/v1/acl/bootstrap

This returns a Consul ACL management token that should be saved as it will be needed in all subsequent Consul API calls.

- Export the CONSUL_HTTP_TOKEN environment variable and specify the token from step #4 as its value.

export CONSUL_HTTP_TOKEN="<TOKEN from #4>"

- Generate the Consul ACL agent Token by first creating a JSON payload for the API.

cat > payload.json <<'EOF'
{
  "Name": "Agent Token",
  "Type": "client",
  "Rules": "node \"\" { policy = \"write\" } service \"\" { policy = \"read\" }"
}
EOF

- Then generate the token.


curl  --request PUT  \
        --header "X-Consul-Token: ${CONSUL_HTTP_TOKEN}" \
        --data @payload.json \
        https://"${CONSUL_HTTP_ADDR}"/v1/acl/create

This returns a Consul agent token.

{"ID":"7f35dea8-4bfa-700e-f34a-7874723f8e2e"}

- Update Consul configuration in /etc/consul.d/consul.hcl on all Consul servers and client agents.


acl_agent_token = "<TOKEN from #7>"

- Generate a Consul ACL token for Vault to use by first creating a JSON payload for the API.

cat > payload.json <<'EOF'
{
  "Name": "Vault Token",
  "Type": "client",
  "Rules":
    "key \"vault/\" { policy = \"write\" } node \"\" { policy = \"write\" } service \"vault\" { policy = \"write\" } agent \"\" { policy = \"write\" } session \"\" { policy = \"write\" }"
}
EOF

- Then generate the token.


curl --request PUT \
      --header "X-Consul-Token: ${CONSUL_HTTP_TOKEN}" \
      --data @payload.json \
      https://"${CONSUL_HTTP_ADDR}"/v1/acl/create

This returns a Consul agent token.

{"ID":"5cdde2da-1107-a0e8-c9b2-bcc08436d09b"}

- Update the Consul storage stanza in the Vault configuration file (/etc/vault.d/vault.hcl) on all Vault servers with the Vault Token.

storage "consul" {
  address = "[consul address]"
  path = "vault/"
  token = "<TOKEN from #10>"
}

Save the token ID for use in the Vault configuration.

