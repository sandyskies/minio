# KMS Quickstart Guide [![Slack](https://slack.minio.io/slack?type=svg)](https://slack.minio.io)

KMS feature allows you to use Vault to generate and manage encryption keys for use by the minio server to encrypt objects. This document explains how to configure Minio with Vault as KMS.

## Get started

### 1. Prerequisites
Install Minio - [Minio Quickstart Guide](https://docs.minio.io/docs/minio-quickstart-guide).

### 2. Configure Vault
Vault as Key Management System requires following to be configured in Vault

- [transit](https://www.vaultproject.io/docs/secrets/transit/index.html) backend configured with a named encryption key-ring
- [AppRole](https://www.vaultproject.io/docs/auth/approle.html) based authentication with read/update policy for transit backend. In particular, read and update policy are required for the [Generate Data Key](https://www.vaultproject.io/api/secret/transit/index.html#generate-data-key) endpoint and [Decrypt Data](https://www.vaultproject.io/api/secret/transit/index.html#decrypt-data) endpoint.

Here is a sample quick start for configuring vault with a transit backend and Approle with correct policy 
#### 2.1 Start Vault server in dev mode
In dev mode, Vault server runs in-memory and starts unsealed. Note that running Vault in dev mode is insecure and any data stored in the Vault is lost upon restart.
```
vault server -dev
```

#### 2.2 Set up vault transit backend and create an app role
```
cat > vaultpolicy.hcl <<EOF
path "transit/datakey/plaintext/my-minio-key" { 
  capabilities = [ "read", "update"]
}
path "transit/decrypt/my-minio-key" { 
  capabilities = [ "read", "update"]
}
path "transit/encrypt/my-minio-key" { 
  capabilities = [ "read", "update"]
}

EOF

export VAULT_ADDR='http://127.0.0.1:8200'
vault auth enable approle    # enable approle style auth
vault secrets enable transit  # enable transit secrets engine
vault write -f  transit/keys/my-minio-key  #define a encryption key-ring for the transit path
vault policy write minio-policy ./vaultpolicy.hcl  #define a policy for AppRole to access transit path
vault write auth/approle/role/my-role token_num_uses=0  secret_id_num_uses=0  period=60s # period indicates it is renewable if token is renewed before the period is over
# define an AppRole
vault write auth/approle/role/my-role policies=minio-policy # apply policy to role
vault read auth/approle/role/my-role/role-id  # get Approle ID
vault write -f auth/approle/role/my-role/secret-id

```

The AppRole ID, AppRole Secret Id, Vault endpoint and Vault key name can now be used to start minio server with Vault as KMS.

Note: If [Vault Namespaces](https://learn.hashicorp.com/vault/operations/namespaces) are in use, VAULT_NAMESPACE variable needs to be set before setting approle and transit secrets engine.

### 3. Environment variables

You'll need the Vault endpoint, AppRole ID, AppRole SecretID and encryption key-ring name defined in step 2.2

```sh
export MINIO_SSE_VAULT_APPROLE_ID=9b56cc08-8258-45d5-24a3-679876769126
export MINIO_SSE_VAULT_APPROLE_SECRET=4e30c52f-13e4-a6f5-0763-d50e8cb4321f
export MINIO_SSE_VAULT_ENDPOINT=https://vault-endpoint-ip:8200
export MINIO_SSE_VAULT_KEY_NAME=my-minio-key
minio server ~/export
```

Optionally set `MINIO_SSE_VAULT_CAPATH` as the path to a directory of PEM-encoded CA cert files to verify the Vault server SSL certificate.
```
export MINIO_SSE_VAULT_CAPATH=/home/user/custom-pems
```

Optionally set `MINIO_SSE_VAULT_NAMESPACE` if AppRole and Transit Secrets engine have been scoped to Vault Namespace
```
export MINIO_SSE_VAULT_NAMESPACE=ns1
```

For S3 gateway, three encryption modes are possible. Encryption can be set to ``pass-through`` to backend, ``single encryption`` (at the gateway) or ``double encryption`` (single encryption at gateway and pass through to backend). This can be specified by setting MINIO_GW_SSE and KMS environment variables set in Step #3.

If MINIO_GW_SSE and KMS are not setup, all encryption headers are passed through to the backend. If KMS environment variables are set up, ``single encryption`` is automatically performed at the gateway and encrypted object is saved at the backend.

To specify ``double encryption``, MINIO_GW_SSE environment variable needs to be set to "s3" for sse-s3
and "c" for sse-c encryption. More than one encryption option can be set, delimited by ";". Objects are encrypted at the gateway and the gateway also does a pass-through to backend. Note that in the case of SSE-C encryption, gateway derives a unique SSE-C key for pass through from the SSE-C client key using a KDF.

```sh
export MINIO_GW_SSE="s3;c"
export MINIO_SSE_VAULT_APPROLE_ID=9b56cc08-8258-45d5-24a3-679876769126
export MINIO_SSE_VAULT_APPROLE_SECRET=4e30c52f-13e4-a6f5-0763-d50e8cb4321f
export MINIO_SSE_VAULT_ENDPOINT=https://vault-endpoint-ip:8200
export MINIO_SSE_VAULT_KEY_NAME=my-minio-key
minio gateway s3
```

### 4. Test your setup

To test this setup, start minio server with environment variables set in Step 3, and server is ready to handle SSE-S3 requests.

# Explore Further

- [Use `mc` with Minio Server](https://docs.minio.io/docs/minio-client-quickstart-guide)
- [Use `aws-cli` with Minio Server](https://docs.minio.io/docs/aws-cli-with-minio)
- [Use `s3cmd` with Minio Server](https://docs.minio.io/docs/s3cmd-with-minio)
- [Use `minio-go` SDK with Minio Server](https://docs.minio.io/docs/golang-client-quickstart-guide)
- [The Minio documentation website](https://docs.minio.io)
