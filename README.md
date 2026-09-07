# Modernize to SQL PaaS

**Author:** Fabrizio Mastrogiovanni

---

## Objective

Replace a self-managed database VM with Azure SQL Database (PaaS), and give the web server access to the database password through a Managed Identity — so no credential is ever stored on the VM or in code.

---

## Watch me on the Walkthrough Video
https://www.loom.com/share/0b2c3d415af444f0b861a5ff28bd5de8

## Architecture

```
┌────────── rg-lab02 ──────────┐   ┌────── rg-lab03-fabrizio ──────┐
│                              │   │                               │
│   ┌──────────────────────┐   │   │  ┌─────────────────────────┐  │
│   │    vm-web-lab02      │   │   │  │  kv-lab03-fabrizio      │  │
│   │                      │   │   │  │  Secret:                │  │
│   │  System-assigned MI  │───┼─①─┼─▶│   SqlAdminPassword      │  │
│   │         │            │   │   │  │  Role: Secrets User     │  │
│   │    IMDS │ 169.254.   │   │   │  │        (read-only)      │  │
│   │         ▼ 169.254    │   │   │  └─────────────────────────┘  │
│   │   Entra ID token     │   │   │                               │
│   │         │            │   │   │  ┌─────────────────────────┐  │
│   │         ▼            │   │   │  │  sql-server-fabrizio    │  │
│   │  password in memory  │───┼─②─┼─▶│  Database: sql-app      │  │
│   └──────────────────────┘   │   │  │  Tier: Basic (5 DTU)    │  │
│                              │   │  └─────────────────────────┘  │
│   ✖ vm-db-lab02 (retired)    │   │                               │
└──────────────────────────────┘   └───────────────────────────────┘

  ① VM → Key Vault    identity-based, no password
  ② VM → Azure SQL    SQL auth, using the password from ①
```

**Key point:** the Managed Identity authenticates to **Key Vault only**, not to SQL. The server uses SQL authentication, so the identity can't reach the database directly. Making that hop identity-based too is listed under Future Work.

---

## Prerequisites

| Need | Check |
|---|---|
| Azure CLI | `brew install azure-cli` → `az version` |
| Web VM running | `az vm show -d -g rg-lab02 -n vm-web-lab02 -o table` |
| **bash** shell | If you use fish, run `bash` first — `VAR="x"` fails in fish |

---

## Variables

```bash
RG_VM="rg-lab02"
RG_LAB03="rg-lab03-fabrizio"
VM="vm-web-lab02"
KV="kv-lab03-fabrizio"
SQL_SERVER="sql-server-fabrizio"
SQL_DB="sql-app"
```

> Confirm the real database name before scripting — the portal auto-generates one and it's easy to accept it by accident:
> `az sql db list -g "$RG_LAB03" --server "$SQL_SERVER" --query "[].name" -o tsv`

---

## Build

### 1 · Provision (portal)

| Setting | Value | Why |
|---|---|---|
| Tier | Basic, DTU-based | ~$5/mo. Dismiss the free-offer banner first or DTU won't appear |
| Workload | Development | Production defaults to Hyperscale at $320+/mo |
| Backup | LRS | Cheapest; production would use ZRS/GRS |
| KV permissions | Azure RBAC | Auditable, consistent across services |
| Purge protection | Disabled | Otherwise the vault can't be deleted for 90 days |

> With RBAC, **you** have no access to your own vault by default. Self-assign `Key Vault Administrator` before creating the secret.

### 2 · Enable the Managed Identity

```bash
PRINCIPAL_ID=$(az vm show -g "$RG_VM" -n "$VM" --query identity.principalId -o tsv)
echo "$PRINCIPAL_ID"    # expect a GUID
```

If empty: `az vm identity assign -g "$RG_VM" -n "$VM"`

### 3 · Grant least-privilege access

```bash
KV_ID=$(az keyvault show -n "$KV" --query id -o tsv)

az role assignment create \
  --assignee-object-id "$PRINCIPAL_ID" \
  --assignee-principal-type ServicePrincipal \
  --role "Key Vault Secrets User" \
  --scope "$KV_ID"
```

| Identity | Role | Can do |
|---|---|---|
| You | Key Vault Administrator | Full manage |
| vm-web-lab02 | Key Vault Secrets User | **Read secrets only** |

> Wait 1–2 minutes. RBAC propagation delay is the #1 cause of a false 403.

### 4 · Install sqlcmd on the VM

Run Command goes through the Azure control plane — no SSH, no NSG change, no dependency on your home IP.

```bash
az vm run-command invoke -g "$RG_VM" -n "$VM" --command-id RunShellScript --scripts '
rm -f /etc/apt/trusted.gpg.d/microsoft.asc
curl -fsSL https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor -o /usr/share/keyrings/microsoft-prod.gpg
curl -fsSL https://packages.microsoft.com/config/ubuntu/24.04/prod.list -o /etc/apt/sources.list.d/mssql-release.list
apt-get update -qq || { echo "FAIL: apt update"; exit 1; }
ACCEPT_EULA=Y apt-get install -y -qq mssql-tools18 unixodbc-dev || { echo "FAIL: install"; exit 1; }
[ -x /opt/mssql-tools18/bin/sqlcmd ] && echo "PASS: sqlcmd installed" || { echo "FAIL: missing"; exit 1; }
'
```

> **Ubuntu 24.04 note:** Microsoft's repo file expects the key at `/usr/share/keyrings/microsoft-prod.gpg` in binary form. Dropping a `.asc` into `trusted.gpg.d/` gives `NO_PUBKEY` / "repository is not signed".

---

## Validation

### 5 · Identity chain — including the denial test

```bash
az vm run-command invoke -g "$RG_VM" -n "$VM" --command-id RunShellScript --scripts '
KV="kv-lab03-fabrizio"

TOKEN=$(curl -s -H "Metadata: true" \
  "http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https%3A%2F%2Fvault.azure.net" \
  | python3 -c "import sys,json; print(json.load(sys.stdin).get(\"access_token\",\"\"))")
[ -n "$TOKEN" ] && echo "PASS: token acquired" || { echo "FAIL: no token"; exit 1; }

CODE=$(curl -s -o /tmp/s.json -w "%{http_code}" -H "Authorization: Bearer $TOKEN" \
  "https://$KV.vault.azure.net/secrets/SqlAdminPassword?api-version=7.4")
VAL=$(python3 -c "import json; print(json.load(open(\"/tmp/s.json\")).get(\"value\",\"\"))" 2>/dev/null)
[ -n "$VAL" ] && echo "PASS: secret read ($CODE, ${#VAL} chars)" || echo "FAIL: empty secret"

WRITE=$(curl -s -o /dev/null -w "%{http_code}" -X PUT \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d "{\"value\":\"nope\"}" "https://$KV.vault.azure.net/secrets/ShouldFail?api-version=7.4")
[ "$WRITE" = "403" ] && echo "PASS: write denied (403)" || echo "REVIEW: write returned $WRITE"

rm -f /tmp/s.json
'
```

**Expected:**
```
PASS: token acquired
PASS: secret read (200, 13 chars)
PASS: write denied (403)
```

**Why this matters:**
- `169.254.169.254` is the Instance Metadata Service — link-local, unroutable, reachable only from inside the VM. Nothing outside can request that token.
- **The 403 is the real test.** A successful read only shows the happy path. Watching the write get refused is what proves the read-only role is an enforced boundary. Same logic as the Lab 02 NSG work: an Allow rule proves nothing until something gets denied.
- Never `echo` the secret — print its length instead, so screenshots stay safe.

### 6 · Database connection

```bash
az vm run-command invoke -g "$RG_VM" -n "$VM" --command-id RunShellScript --scripts '
KV="kv-lab03-fabrizio"
SQL="sql-server-fabrizio.database.windows.net"

TOKEN=$(curl -s -H "Metadata: true" \
  "http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https%3A%2F%2Fvault.azure.net" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)[\"access_token\"])")

PWD_SQL=$(curl -s -H "Authorization: Bearer $TOKEN" \
  "https://$KV.vault.azure.net/secrets/SqlAdminPassword?api-version=7.4" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)[\"value\"])")
echo "PASS: secret retrieved (${#PWD_SQL} chars)"

/opt/mssql-tools18/bin/sqlcmd -S "tcp:$SQL,1433" -d sql-app \
  -U sqladmin -P "$PWD_SQL" -N -C -l 30 \
  -Q "SELECT DB_NAME() AS db, SUSER_NAME() AS login, GETUTCDATE() AS utc;" \
  && echo "PASS: SQL connection successful" || echo "FAIL: SQL connection"
'
```

**Expected:**
```
PASS: secret retrieved (13 chars)

db        login      utc
--------- ---------- -----------------------
sql-app   sqladmin   2026-09-07 17:21:51.943

(1 rows affected)
PASS: SQL connection successful
```
### 7 · Observability

Portal → `sql-app` → **Monitoring → Metrics** → `DTU percentage`, aggregation `Max`. A near-zero line still confirms the database is live and emitting telemetry. Sustained 80%+ is the scale-up signal in production.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `Unknown command: RG_VM` | fish shell | Run `bash` first |
| Empty `principalId` | MI not enabled | `az vm identity assign` |
| Key Vault **403** | RBAC not propagated | Wait 2 min, verify assignment |
| Key Vault **404** | Wrong secret name | `az keyvault secret list --vault-name $KV -o table` |
| `NO_PUBKEY` on apt | Ubuntu 24.04 keyring path | `gpg --dearmor` (step 4) |
| `Cannot open database "X"` | **Wrong DB name** — auth worked | `az sql db list ...`, fix `-d` |
| `Login failed` alone | Password mismatch | Rotate (step 7) to force sync |
| sqlcmd timeout | SQL firewall | See below |

**Read the SQL error carefully.** `Cannot open database` means the firewall allowed you in, TLS negotiated, and login succeeded — only the database lookup failed. A bare `Login failed for user` would point at the credential instead. Knowing which layer failed turns a 30-minute hunt into a 30-second fix.

**Firewall:**
```bash
az sql server firewall-rule list -g "$RG_LAB03" --server "$SQL_SERVER" -o table
```
A rule with start and end IP `0.0.0.0` must exist. That's not "open to the internet" — it's a sentinel value meaning *traffic originating inside Azure*.

---

## Notes on the Source Lab

Three issues found and corrected:

1. **Checklist contradicts the build.** It asks for a Vault Access Policy with Get/List — the legacy model. The lab configures RBAC. Corrected to the role assignment.
2. **Ubuntu 24.04 install path is broken** (see step 4).
3. **No end-to-end validation.** The lab stops at "the role assignment exists," which confirms configuration, not function. Steps 5–7 close that gap.

**Intentional deviation:** `vm-db-lab02` was kept, not deleted, to preserve the Lab 02 NSG rules (`Allow-Web-To-Postgres` @100, `Deny-All-VNet-Inbound` @4000) as demonstrable evidence.

---

## Teardown

```bash
az group delete -n "$RG_LAB03" --yes --no-wait
az group exists -n "$RG_LAB03"    # false
```

Purge protection was left off so the vault deletes permanently instead of sitting in soft-delete for 90 days. If it lingers: `az keyvault purge -n "$KV"`.

**Keep `rg-lab02`** if Lab 04 needs the web VM or the NSG evidence.

---

## Summary

A web server needs a database password. Putting it in a file on the server works, but now it can be copied, committed to git, or read by anyone with server access — and changing it means touching every machine that has a copy.

This lab moves the password into Azure Key Vault and gives the web server a way to prove who it is without holding any credential. Azure issues the VM an identity; the VM asks a link-local address that only exists inside itself for a short-lived token, hands that token to Key Vault, gets the password, uses it, and never writes it down.

Two things make it more than a config exercise. The identity was given read-only access and that was **verified by trying to write and getting a 403** — an unenforced permission isn't a boundary. Then the password was **rotated with the app still working and nothing on the server touched**, which is the practical argument for the whole design.

The database VM was also replaced with Azure SQL Database, moving patching, backups, and availability from a maintenance task to a platform responsibility.

**Skills:** IaaS→PaaS migration · Key Vault · Managed Identity · Azure RBAC & least privilege · credential rotation · IMDS/Entra token auth · agentless validation via the control plane · Azure Monitor
