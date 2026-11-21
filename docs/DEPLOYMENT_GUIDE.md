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

See the [mainnet deployment runbook](../scripts/MAINNET_DEPLOYMENT_RUNBOOK.txt)
and [upgrade migration guide](UPGRADE_MIGRATION.md) for deployment and upgrade
procedures.
