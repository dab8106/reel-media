# Install HashiCorp Vault the right way (single node + first setup)

Companion guide for the dabeer.ai video "Install HashiCorp Vault Step by Step" (YouTube @dabeer.ai). Every command below was run end to end on a fresh Ubuntu 24.04 machine with Vault 2.2.1 (November 2026).

**You will end up with:** Vault installed from HashiCorp's signed repository, running as a systemd service with TLS and Raft (integrated) storage, initialized and unsealed, with an audit log, a KV secrets engine, an admin login with a policy, the root token revoked, and a first backup.

> Never run `vault server -dev` on a real server. Dev mode keeps everything in memory, starts unsealed and prints a root token. It is for your laptop only.

## 0. What you need

- Ubuntu 22.04 or 24.04 server, and a user that can run `sudo`.
- A DNS name for the server. This guide uses `vault.example.com`: replace it everywhere.
- Ports **8200** (API and web UI) and **8201** (cluster traffic) open **only to your own network**, in your firewall or cloud security group.

## 1. Install from HashiCorp's signed repository

```bash
sudo apt update && sudo apt install -y gpg wget lsb-release

wget -qO- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

# Compare this fingerprint with the one HashiCorp publishes on its website
gpg --show-keys /usr/share/keyrings/hashicorp-archive-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt update && sudo apt install -y vault
vault version
```

The package creates the `vault` user, `/etc/vault.d/vault.hcl`, `/opt/vault/data`, a test certificate in `/opt/vault/tls`, and a hardened `vault.service`.

## 2. Prepare the server

```bash
# Turn swap off now, and keep it off after a reboot
sudo swapoff -a && sudo sed -i "/\sswap\s/ s/^/#/" /etc/fstab

# Folder for the audit log (enabling audit fails without it)
sudo mkdir -p /var/log/vault && sudo chown vault:vault /var/log/vault

# Only Vault may read its data folder
sudo chmod 700 /opt/vault/data
```

## 3. TLS certificate

The package's test certificate has no host name in it (`CN=Vault`), so clients reject it. Replace it.

**Option A, your company CA (production):** request a certificate for `vault.example.com` (include the full chain in the .crt), then copy it as `vault.crt` / `vault.key` and go to "Install it" below.

**Option B, a small lab CA:**

```bash
openssl req -x509 -newkey rsa:4096 -sha256 -days 365 -nodes -keyout ca.key -out ca.crt -subj "/CN=Lab Vault CA"

openssl req -newkey rsa:4096 -nodes -keyout vault.key -out vault.csr -subj "/CN=vault.example.com" -addext "subjectAltName=DNS:vault.example.com,IP:127.0.0.1"

openssl x509 -req -in vault.csr -CA ca.crt -CAkey ca.key -CAcreateserial -days 365 -copy_extensions copy -out vault.crt
```

**Install it:**

```bash
sudo install -o vault -g vault -m 644 vault.crt /opt/vault/tls/tls.crt
sudo install -o vault -g vault -m 600 vault.key /opt/vault/tls/tls.key

# Lab CA only: trust it on every machine that talks to Vault
sudo cp ca.crt /usr/local/share/ca-certificates/lab-vault-ca.crt && sudo update-ca-certificates

# Check: the DNS name must be in the certificate
openssl x509 -in vault.crt -noout -subject -ext subjectAltName
```

Keep `ca.key` offline; you only need it to issue certificates.

## 4. Configuration

```bash
sudo cp /etc/vault.d/vault.hcl /etc/vault.d/vault.hcl.orig

sudo tee /etc/vault.d/vault.hcl > /dev/null <<'EOF'
ui            = true
disable_mlock = true

api_addr     = "https://vault.example.com:8200"
cluster_addr = "https://vault.example.com:8201"

storage "raft" {
  path    = "/opt/vault/data"
  node_id = "vault-1"
}

listener "tcp" {
  address       = "0.0.0.0:8200"
  tls_cert_file = "/opt/vault/tls/tls.crt"
  tls_key_file  = "/opt/vault/tls/tls.key"
}
EOF

sudo chown vault:vault /etc/vault.d/vault.hcl && sudo chmod 640 /etc/vault.d/vault.hcl

# Checks config, TLS and storage before you start
sudo -u vault vault operator diagnose -config=/etc/vault.d/vault.hcl
```

- **Raft storage**: data stays inside Vault, no separate database, and it can grow into a 3- or 5-node cluster.
- **`disable_mlock = true`**: HashiCorp's recommendation with Raft storage, together with swap off (step 2).
- On a single node, the diagnose warnings "Check Telemetry" and "Check For Raft Quorum" are expected.

## 5. Start Vault

```bash
sudo systemctl enable --now vault
sudo systemctl status vault --no-pager      # look for: Active: active (running)
sudo journalctl -u vault -n 50 --no-pager   # if it isn't running, the reason is here

# Tell the vault CLI where the server is (and keep it set)
echo "export VAULT_ADDR=https://vault.example.com:8200" >> ~/.bashrc && source ~/.bashrc

vault status                                 # Initialized false, Sealed true = expected
```

## 6. Initialize and unseal

```bash
vault operator init
```

- Vault splits its root key into **5 unseal key shares; any 3 unseal it**, and prints an **initial root token**.
- **This output is shown only once.** Save it in a password manager now. In a team, give each share to a different person (or encrypt each share for its holder with `-pgp-keys`).
- Lose more than 2 shares and the data cannot be recovered.

```bash
vault operator unseal   # share 1 (input is hidden)  -> Unseal Progress 1/3
vault operator unseal   # share 2                    -> 2/3
vault operator unseal   # share 3                    -> Sealed false
```

Every restart seals Vault again. For production, look at auto unseal with a cloud key service (AWS KMS, Azure Key Vault, Google Cloud KMS).

## 7. First setup

```bash
vault login                       # paste the initial root token (input is hidden)

vault audit enable file file_path=/var/log/vault/vault_audit.log

vault secrets enable -path=secret kv-v2
vault kv put secret/myapp/db username=app password=Change-Me-123
vault kv get secret/myapp/db
```

Create `admin.hcl` (day-to-day admin without root):

```hcl
# Day-to-day admin without root
path "sys/auth"                       { capabilities = ["read"] }
path "sys/auth/*"                     { capabilities = ["create", "read", "update", "delete", "sudo"] }
path "sys/mounts"                     { capabilities = ["read"] }
path "sys/mounts/*"                   { capabilities = ["create", "read", "update", "delete", "list"] }
path "sys/policies/acl/*"             { capabilities = ["create", "read", "update", "delete", "list"] }
path "sys/audit*"                     { capabilities = ["create", "read", "update", "delete", "list", "sudo"] }
path "auth/*"                         { capabilities = ["create", "read", "update", "delete", "list"] }
path "secret/*"                       { capabilities = ["create", "read", "update", "delete", "list"] }
path "sys/storage/raft/snapshot"      { capabilities = ["read"] }
path "sys/storage/raft/configuration" { capabilities = ["read"] }
```

```bash
vault policy write admin admin.hcl

vault auth enable userpass
vault write auth/userpass/users/admin1 password='use-a-long-passphrase' policies=admin
# In a company, use your single sign-on instead (OIDC or LDAP auth methods)

vault login -method=userpass username=admin1      # token_policies ["admin" "default"]

# Revoke the root token (replace <root-token>), then check it is gone
VAULT_TOKEN=<root-token> vault token revoke -self
VAULT_TOKEN=<root-token> vault token lookup        # 403 permission denied, invalid token

vault operator raft snapshot save vault-backup.snap
```

- Need root again later? A quorum of key holders can create one with `vault operator generate-root`.
- Run the snapshot on a schedule and store the file **off the server**.
- Web UI: `https://vault.example.com:8200/ui`, sign in with method **Userpass**.

## Checklist

- [ ] Installed from the signed HashiCorp repository
- [ ] Swap off, data folder 700, audit folder owned by `vault`
- [ ] Real TLS certificate with your DNS name
- [ ] Raft storage, `vault operator diagnose` passes
- [ ] Unseal shares stored apart (or auto unseal)
- [ ] Audit device enabled
- [ ] Admin login through a policy; root token revoked
- [ ] Scheduled snapshots stored off the server

**Next:** connect an application to Vault, so it reads secrets without a password in the code.
