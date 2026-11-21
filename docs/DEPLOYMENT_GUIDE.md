# Deployment Guide: Identifying the Deployed WASM

This guide explains how an operator traces an on-chain deployment back to the
exact contract bytecode. It is also the audit trail to use before approving an
upgrade.

## 1. Locate the Release Artifact

For a CI-built release, open the successful GitHub Actions run for the release
tag or source commit. In the **Build WASM** job, download the
`neurowealth-vault-wasm` artifact. The archive contains:

```text
neurowealth_vault.wasm
```

The CI workflow produces this artifact in
[`.github/workflows/ci.yml`](../.github/workflows/ci.yml). It retains uploaded
artifacts for 14 days, so copy the archive and checksum into the release ticket
before that retention period expires.

For a local or devnet build, the conventional artifact location is:

```bash
neurowealth-vault/contracts/target/wasm32-unknown-unknown/release/neurowealth_vault.wasm
```

If the file came from a devnet E2E run, also retain
`scripts/e2e-artifacts/contract_id.txt` and
`scripts/e2e-artifacts/wasm_hash.txt`. Mainnet deployments are recorded in
`scripts/mainnet-deployment-artifacts.json`.

## 2. Read and Record the Artifact Checksum

Compute a SHA-256 checksum directly from the downloaded artifact:

```bash
sha256sum neurowealth_vault.wasm
# macOS / systems without sha256sum:
shasum -a 256 neurowealth_vault.wasm
```

Record the 64-character hexadecimal value by itself, without the filename.
Examples and format details are:

```json
{
  "wasm_hash": "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef",
  "artifact_sha256": "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef"
}
```

`artifact_sha256` is the checksum of the downloaded bytes. `wasm_hash` is the
same checksum normalized for comparison with Stellar/Soroban metadata. Also
record the artifact name, source commit or release tag, GitHub Actions job URL,
and CI run ID in the deployment ticket.

## 3. Rebuild and Verify the Bytes (When Required)

For a stronger audit, rebuild the exact source revision from clean source:

```bash
git checkout <release-tag-or-commit>
./scripts/check-reproducible-build.sh
```

The reproducibility script prints SHA-256 values for builds made in two isolated
directory roots. Their hashes must match each other and the recorded release
artifact. Do not proceed if they differ.

## 4. Compare Published Deployment Metadata

The mainnet deployment record is
[`scripts/mainnet-deployment-artifacts.json`](../scripts/mainnet-deployment-artifacts.json).
It should contain the following WASM fields:

| Field | Meaning |
|---|---|
| `wasm_hash` | Deployed/uploaded WASM SHA-256 hash |
| `artifact_sha256` | SHA-256 of the uploaded release artifact |
| `wasm_artifact` | Artifact name or stable artifact URL |
| `source_commit` | Commit from which the WASM was built |
| `release_tag` | Optional signed release tag |
| `ci_job_url` | Link to the exact CI build job |

Read the record and compare its values:

```bash
jq '{network, vault_contract_id, deployment_tx_hash, wasm_hash,
     artifact_sha256, wasm_artifact, source_commit, release_tag,
     ci_job_url}' scripts/mainnet-deployment-artifacts.json

ARTIFACT_HASH=$(sha256sum neurowealth_vault.wasm | awk '{print $1}')
RECORDED_HASH=$(jq -r '.artifact_sha256' scripts/mainnet-deployment-artifacts.json)

test "$ARTIFACT_HASH" = "$RECORDED_HASH"
```

The final command exits with status 0 only if the release artifact and
deployment metadata name the same bytes.

## 5. Compare with Stellar

Installing or uploading the WASM on Stellar/Soroban should return the contract
WASM hash:

```bash
stellar contract install \
  --network mainnet \
  --source-account "$DEPLOYER_ADDRESS" \
  --wasm neurowealth_vault.wasm
```

The returned value must be a 64-character lowercase hex SHA-256 and must equal:

```text
downloaded artifact checksum
= scripts/mainnet-deployment-artifacts.json artifact_sha256
= scripts/mainnet-deployment-artifacts.json wasm_hash
= Stellar contract install/upload return value
```

For an upgrade, query the pending proposal and compare the scheduled hash:

```bash
stellar contract invoke \
  --id "$VAULT_CONTRACT_ID" \
  --network mainnet \
  -- \
  get_pending_upgrade
```

The `wasm_hash` returned in the pending upgrade tuple must match the same
recorded release hash before the upgrade is approved or executed.

Verify the installed contract ID and deployment transaction in Stellar Expert
against `vault_contract_id` and `deployment_tx_hash` in the metadata:

```text
https://stellar.expert/explorer/public/contract/<vault_contract_id>
https://stellar.expert/explorer/public/tx/<deployment_tx_hash>
```

## 6. Stop Conditions

Treat identification as failed and stop the deployment or upgrade if any of
these differ:

- Artifact checksum and published deployment metadata
- Metadata `wasm_hash` and Stellar install/upload result
- Pending upgrade hash and approved release hash
- Release source commit/tag and the CI job that produced the artifact
- Contract ID or deployment transaction and the published deployment record

Never substitute a locally rebuilt, unreviewed, or differently named WASM when
the hashes do not match. Preserve the downloaded artifact, checksum, metadata
JSON, CI run URL, and install/upload output as the release audit evidence.
