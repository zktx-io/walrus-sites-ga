# Walrus Sites GA — Provenance Workflow

A reusable GitHub Actions workflow that **extends the official Walrus Sites pipeline with provenance support**.
It builds your static site, generates [SLSA](https://slsa.dev)-compliant provenance, verifies integrity for every file, and deploys to Walrus.

* ✅ Full-chain: build → provenance (SLSA3) → verify → deploy
* ✅ Supports Mainnet & Testnet
* ✅ Publishes `.well-known/walrus-sites.intoto.jsonl` automatically
* ✅ Reusable `workflow_call` design for team-wide adoption

---

## Prerequisites

* **Sui address**: `SUI_ADDRESS` (set as environment variable or org variable)
* **Keystore**: `SUI_KEYSTORE` (store in GitHub Secrets)
* **`site.config.json`** file in repo root:

```json
{
  "path": "dist",          // Build output directory (required)
  "network": "testnet",    // "mainnet" or "testnet"
  "epochs": 5              // Retention period in epochs
}
```

> ⚠️ The `path` folder **must not already exist** when workflow starts.

---

## Credentials

* `SUI_KEYSTORE` (Secret): must be formatted as a JSON array with one string element, e.g.:

```json
["AXXXXXXXXXXXXXXXXXXXXXXXXX"]
```

* `SUI_ADDRESS` (Variable): the matching Sui address, e.g. `0x123...abc`.

> ⚠️ Use a dedicated Sui address per workflow to avoid gas conflicts.
> Ensure the address is funded with **SUI** (gas fees) and **WAL** (storage).

---

## Initial Deployment

Run the workflow once to deploy your site with provenance.

```yaml
name: Deploy Walrus Site with Provenance (one-off)

on:
  workflow_dispatch:

permissions:
  id-token: write
  contents: write
  actions: read

jobs:
  deploy-with-provenance:
    uses: zktx-io/walrus-sites-ga/.github/workflows/deploy_with_slsa3.yml@v0.4.7
    with:
      network: 'testnet'                 # or 'mainnet'
      epochs: 5
      SUI_ADDRESS: ${{ vars.SUI_ADDRESS }}
    secrets:
      SUI_KEYSTORE: ${{ secrets.SUI_KEYSTORE }}
```

---

## Update the Site

To update an existing Walrus Site:

1. The workflow reads the **site object ID** from `ws-resources.json` in your deploy root, so you don’t need to pass it manually.
2. Only modified resources are updated, while routes and unchanged files stay intact.

---

## About `ws-resources.json`

* **First deployment:** not required
* **Updates:** required — must exist in `DIST/` so the workflow can update the same site.
* **Location:** defaults to `DIST/ws-resources.json`. Place it under `public/` in your framework so it auto-copies into `DIST/`.

When preparing for updates, make sure to include:

* `object_id` (from the initial deployment)
* `site_name`
* `metadata.link`
* `metadata.description`
* `metadata.project_url`

Example:

```json
{
  "object_id": "0x1234abcd...",
  "site_name": "My Walrus Site",
  "metadata": {
    "link": "https://example.com",
    "description": "Demo site deployed on Walrus",
    "project_url": "https://github.com/example/repo"
  }
}
```

> If `GITHUB_TOKEN` is configured, deployments that update this file will open a PR.
> **Merge the PR** so the `object_id` is stored in your repo and future runs keep updating the same site.

---

## What this workflow does

1. **Build**: Runs `npm ci && npm run build` (Node 20)
2. **Hash**: Computes SHA-256 for all build artifacts (Base64-encoded list)
3. **Provenance (SLSA3)**: Generates `walrus-sites.intoto.jsonl` via official SLSA generator
4. **Verify**: Uses `slsa-verifier` to check every file against provenance
5. **Deploy**: Publishes artifacts + `.well-known/walrus-sites.intoto.jsonl` to Walrus

---

## Inputs

| Input               | Type   | Required | Default | Description                    |
| ------------------- | ------ | -------- | ------- | ------------------------------ |
| `working-directory` | string |          | `.`     | Optional subdirectory to build |
| `network`           | string | ✔        | testnet | `mainnet` or `testnet`         |
| `epochs`            | number | ✔        | 5       | Retention in epochs            |
| `SUI_ADDRESS`       | string | ✔        |         | Deployment wallet address      |

## Secrets

| Secret         | Required | Description                      |
| -------------- | -------- | -------------------------------- |
| `SUI_KEYSTORE` | ✔        | Sui keystore (JSON) — in Secrets |

## Optional Inputs

* `DIST`: Path to your built site directory (required if not using `site.config.json`)
* `WALRUS_CONFIG`: Custom Walrus configuration (default auto-download)
* `SITES_CONFIG`: Custom sites configuration (default auto-download)
* `WS_RESOURCES`: Path to `ws-resources.json` (defaults to `DIST/ws-resources.json`)
* `GITHUB_TOKEN`: Enables PRs when `ws-resources.json` changes

If you want PRs for updated resources, add:

```yaml
permissions:
  contents: write
  pull-requests: write
```

---

## Verify Provenance

After deployment, provenance is published as `.well-known/walrus-sites.intoto.jsonl`.
You can audit deployments with **Walrus Sites Notary**:
👉 [https://notary.wal.app/site/workflow](https://notary.wal.app/site/workflow)

---

## Security Notes

* **Secrets in GitHub**: Convenient, but always risky. Use org secrets/variables and minimize scope.
* **Action pinning**: Pin all third-party actions to commit SHA.
* **Reproducible builds**: Lock dependencies, pin Docker digests, ensure consistent environments.
* **Dedicated keys**: Use a separate Sui address for each workflow to avoid gas conflicts.
* **Funding**: Ensure your deployment address has both SUI (gas fees) and WAL (storage).

---

## Troubleshooting

* **“Output folder already exists”**

  * Ensure the `path` directory doesn’t exist before build.
* **Provenance verification failed**

  * Likely due to build reproducibility issues. Re-run with locked dependencies.
* **Invalid network/address**

  * Double-check `network`, `SUI_ADDRESS`, and `SUI_KEYSTORE` format.

---

## Example

See example repo with release-triggered deployments:
🔗 [https://github.com/zktx-io/walrus-sites-ga-example](https://github.com/zktx-io/walrus-sites-ga-example)

