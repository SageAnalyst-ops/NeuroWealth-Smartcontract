# Deployment Guide: Identifying the Deployed WASM

Use the release build artifact and the deployment record together to identify
the exact vault WASM installed on a network. A matching checksum establishes
that the artifact bytes match the recorded hash; the deployment transaction
and contract address establish that this hash was actually installed for the
deployment being investigated.

## 1. Locate and download the CI artifact

1. Open the GitHub Actions **CI** run for the source commit recorded for the
   deployment. Confirm the run completed successfully and that its source
   commit matches the commit the release tag points to. `ci.yml` builds on
   branch pushes; it is not configured to trigger on tag pushes.
2. In that run, download the **`neurowealth-vault-wasm`** artifact. CI builds
   the optimized `neurowealth_vault.wasm` and retains the artifact for 14 days.
   The artifact contains both the WASM and its `.sha256` sidecar.

Alternatively, with GitHub CLI authenticated, list runs and download from the
matching run ID:

```bash
gh run list --workflow ci.yml --branch main
gh run download RUN_ID --name neurowealth-vault-wasm --dir release-artifact
```

Replace `RUN_ID` with the run for the exact release commit; do not select a
different run just because its branch or tag name looks similar. For a release
on another branch, use that branch in `gh run list`.

## 2. Read and verify the checksum

The `.sha256` file contains the SHA-256 hex digest and WASM filename. From the
directory containing both downloaded files:

```bash
sha256sum -c neurowealth_vault.wasm.sha256
```

`OK` means the downloaded WASM matches its sidecar. The same digest is printed
in the **WASM Size Report** in the CI run summary. On Windows PowerShell, read
the first field in the sidecar and independently compute it with:

```powershell
(Get-Content .\neurowealth_vault.wasm.sha256).Split()[0]
Get-FileHash .\neurowealth_vault.wasm -Algorithm SHA256
```

The `Hash` value from `Get-FileHash` must equal the sidecar value (case
insensitive). If the sidecar is missing or the values differ, stop: do not
identify or deploy the artifact based on its filename alone.

## 3. Compare it with deployment metadata

Find the deployment record for the same network and vault contract. The
mainnet runbook keeps this record at
`scripts/mainnet-deployment-artifacts.json`; deployments are recorded manually,
so do not assume the file exists in the source checkout or that a dry-run
created it. The record should tie together at least:

- network and vault contract ID;
- release tag and source commit;
- CI run URL (or run ID);
- the `neurowealth-vault-wasm` artifact name and `wasm_sha256`;
- the WASM hash returned by the network's contract-install operation;
- the install/deploy transaction hash and deployment time.

Compare the verified sidecar digest with the record's `wasm_sha256` and the
installed WASM hash. Both must match the artifact's 64-character SHA-256
hex digest exactly (hex letter case is insignificant). Also confirm the
record's commit and CI run identify the artifact you downloaded and that its
network and contract ID are the deployment under review. The CI run summary's
SHA-256 is useful corroboration, but is not a substitute for the deployment
record or the on-network install transaction.

For an upgrade, a pending `get_pending_upgrade` value or
`UpgradeScheduledEvent` only proves that a hash was proposed. Verify the
successful `execute_upgrade` transaction after the timelock and match the
installed WASM hash in the deployment record; a scheduled or cancelled hash
alone does not identify the code currently running.

If GitHub's 14-day artifact retention has expired, retrieve the artifact and
deployment record from the team's retained release archive. If neither is
available, the deployed build cannot be established from the artifact name
or source tag alone; record the evidence gap rather than guessing.

---

## Deployment record schema

`scripts/mainnet-deployment-artifacts.json` is populated manually by the
operator who ran the deployment, following the mainnet runbook. Each top-level
key is a vault contract ID (the `C...` address on Stellar mainnet). A minimal
but complete record looks like this:

```json
{
  "CEXAMPLEVAULTCONTRACTID000000000000000000000000000000000": {
    "network": "mainnet",
    "vault_contract_id": "CEXAMPLEVAULTCONTRACTID000000000000000000000000000000000",
    "release_tag": "v1.2.0",
    "source_commit": "a3f8c1d2e4b57609f8e1234abcd5678ef9012345",
    "ci_run_url": "https://github.com/org/repo/actions/runs/9876543210",
    "ci_run_id": "9876543210",
    "artifact_name": "neurowealth-vault-wasm",
    "wasm_sha256": "b94d27b9934d3e08a52e52d7da7dabfac484efe04294e576b7a1d1f0ea7c0a7c",
    "installed_wasm_hash": "b94d27b9934d3e08a52e52d7da7dabfac484efe04294e576b7a1d1f0ea7c0a7c",
    "deploy_tx_hash": "a1b2c3d4e5f6...",
    "deployed_at": "2024-11-15T14:32:00Z",
    "deployed_by": "alice@example.com",
    "notes": "Initial mainnet deployment for v1.2.0"
  }
}
```

Key fields for artifact identification:

| Field | Where it comes from | What to match against |
|---|---|---|
| `source_commit` | Git tag or release branch HEAD | The commit shown in the CI run summary |
| `ci_run_id` | GitHub Actions URL | The run from which you downloaded the artifact |
| `artifact_name` | GitHub Actions artifact name | Must be `neurowealth-vault-wasm` |
| `wasm_sha256` | Recorded at deploy time from the sidecar | The digest in `neurowealth_vault.wasm.sha256` |
| `installed_wasm_hash` | Returned by the Stellar `contract install` command | Must equal `wasm_sha256` |
| `deploy_tx_hash` | Transaction hash from the deploy step | Verifiable on Stellar Expert |

`wasm_sha256` and `installed_wasm_hash` must be identical. If they differ, the
artifact on record was not what was installed; treat this as an integrity
failure and escalate before proceeding.

---

## End-to-end walkthrough

The following example uses shell variables to make each comparison explicit.
Substitute the values from your actual deployment record.

### Step 1 — Export values from the deployment record

```bash
# Read the deployment record for the vault under review.
RECORD=$(cat scripts/mainnet-deployment-artifacts.json)

VAULT_ID="CEXAMPLEVAULTCONTRACTID000000000000000000000000000000000"
RECORD_COMMIT=$(echo "$RECORD" | python3 -c "import json,sys; r=json.load(sys.stdin); print(r['$VAULT_ID']['source_commit'])")
RECORD_RUN_ID=$(echo "$RECORD" | python3 -c "import json,sys; r=json.load(sys.stdin); print(r['$VAULT_ID']['ci_run_id'])")
RECORD_SHA256=$(echo "$RECORD" | python3 -c "import json,sys; r=json.load(sys.stdin); print(r['$VAULT_ID']['wasm_sha256'])")
INSTALLED_SHA256=$(echo "$RECORD" | python3 -c "import json,sys; r=json.load(sys.stdin); print(r['$VAULT_ID']['installed_wasm_hash'])")

echo "Commit  : $RECORD_COMMIT"
echo "CI run  : $RECORD_RUN_ID"
echo "Expected SHA-256: $RECORD_SHA256"
```

### Step 2 — Download the artifact from the matching CI run

```bash
gh run download "$RECORD_RUN_ID" \
  --name neurowealth-vault-wasm \
  --dir release-artifact

ls release-artifact/
# neurowealth_vault.wasm
# neurowealth_vault.wasm.sha256
```

Confirm the CI run's source commit matches `RECORD_COMMIT`:

```bash
gh run view "$RECORD_RUN_ID" --json headSha -q '.headSha'
# Must equal $RECORD_COMMIT
```

### Step 3 — Read the sidecar digest

```bash
SIDECAR_SHA256=$(awk '{print $1}' release-artifact/neurowealth_vault.wasm.sha256)
echo "Sidecar SHA-256: $SIDECAR_SHA256"
```

### Step 4 — Verify the downloaded WASM against the sidecar

```bash
# Linux / macOS
(cd release-artifact && sha256sum -c neurowealth_vault.wasm.sha256)
# Expected output: neurowealth_vault.wasm: OK
```

On Windows PowerShell:

```powershell
$sidecar = (Get-Content release-artifact\neurowealth_vault.wasm.sha256).Split()[0]
$actual  = (Get-FileHash release-artifact\neurowealth_vault.wasm -Algorithm SHA256).Hash.ToLower()
if ($sidecar -eq $actual) { "OK" } else { "MISMATCH — do not proceed" }
```

### Step 5 — Compare against the deployment record

```bash
# Case-insensitive comparison (hex digests may differ in letter case).
SIDECAR_LC=$(echo "$SIDECAR_SHA256" | tr '[:upper:]' '[:lower:]')
RECORD_LC=$(echo "$RECORD_SHA256"   | tr '[:upper:]' '[:lower:]')
INSTALLED_LC=$(echo "$INSTALLED_SHA256" | tr '[:upper:]' '[:lower:]')

if [[ "$SIDECAR_LC" == "$RECORD_LC" && "$SIDECAR_LC" == "$INSTALLED_LC" ]]; then
  echo "PASS: artifact digest matches deployment record and on-chain install."
else
  echo "FAIL:"
  echo "  sidecar   : $SIDECAR_LC"
  echo "  record    : $RECORD_LC"
  echo "  installed : $INSTALLED_LC"
  echo "Do not proceed — investigate the mismatch before any further action."
fi
```

All three values — the downloaded artifact's sidecar digest, `wasm_sha256` in
the deployment record, and `installed_wasm_hash` — must be the same 64-character
hex string. Any discrepancy means either the wrong artifact was downloaded, the
record was not filled in accurately, or the bytes installed on-chain differ from
the release build.

### Step 6 — Cross-check the CI run summary (optional corroboration)

The **WASM Size Report** table in the CI run's job summary also lists the
SHA-256. This is a convenient cross-reference but is not a substitute for the
sidecar or the deployment record — the summary is part of the same CI run and
therefore subject to the same trust boundary.

```bash
# View the step summary for the build-wasm job in the matching run.
gh run view "$RECORD_RUN_ID" --log | grep "SHA-256"
```

The printed digest must equal `$SIDECAR_LC`.

---

## Upgrade deployments

For an upgrade, the same three-way comparison applies, but use the
`execute_upgrade` transaction rather than the initial `contract install`
transaction. A `get_pending_upgrade` hash or an `UpgradeScheduledEvent` only
shows what was *proposed* during the timelock window; it does not confirm what
is *running*. Confirm the upgrade by:

1. Locating the successful `execute_upgrade` transaction hash in the
   deployment record (a separate entry or an `upgrades` array within the
   contract's record).
2. Verifying that transaction on Stellar Expert to confirm it completed after
   the 24-hour timelock and that the WASM hash it installed matches the
   sidecar digest from the new release artifact.

---

## Artifact retention and fallback

GitHub retains the `neurowealth-vault-wasm` artifact for 14 days. After that,
retrieve the artifact from the team's retained release archive. If neither
source is available, the deployed build cannot be established from the artifact
name or the source tag alone — record the evidence gap rather than guessing,
and escalate to the security team.

See the [mainnet deployment runbook](../scripts/MAINNET_DEPLOYMENT_RUNBOOK.txt)
and [upgrade migration guide](UPGRADE_MIGRATION.md) for deployment and upgrade
procedures.
