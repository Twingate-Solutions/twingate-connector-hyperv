# Twingate Connector — Hyper-V Deployment Scripts

> ⚠️ **Example project — provided as-is, with no support or warranty.** These scripts are published as a reference example to build from, not a supported product. Nothing here is guaranteed and no support is attached to it. They were developed with help from an LLM-based coding assistant. Review the code and test it yourself before using it in any critical or production environment. Your use is governed by the Apache License 2.0, including its "AS IS", no-warranty (Section 7), and limitation-of-liability (Section 8) terms.

Automate the deployment and lifecycle management of Twingate Connectors on Windows Server using Hyper-V.

| Script | Purpose |
|---|---|
| [`Deploy-TwingateConnector.ps1`](Deploy-TwingateConnector.ps1) | Deploy and manage Connector VMs end-to-end via the Twingate API |
| [`Reset-TwingateConnectorEnvironment.ps1`](Reset-TwingateConnectorEnvironment.ps1) | Tear down all Connector VMs and the vSwitch after a failed or interrupted run |

> An older deployment method based on a pre-built Ubuntu 22.04 disk image is retained in [`legacy-hyperv-deployment/`](legacy-hyperv-deployment/). It is deprecated and requires manual token entry — use `Deploy-TwingateConnector.ps1` unless you have a specific reason not to.

---

## Deploy-TwingateConnector.ps1

### Overview

Creates Twingate Connectors end-to-end — Twingate API connector records, Ubuntu 24.04 Gen2 Hyper-V VMs, and cloud-init provisioning — with no manual steps beyond running the script. Supports six lifecycle actions: **Deploy**, **Remove**, **UpdateConnector**, **UpdateOS**, **List**, and **FixVM**.

### Prerequisites

- Windows Server 2022 or 2025
- Hyper-V role installed (script can install it and prompt for reboot)
- PowerShell 5.1+ running **as Administrator**
- Internet access (downloads Ubuntu cloud image and qemu-img.exe on first run)
- A Twingate API token with **Read, Write & Provision** scope
- A Remote Network already created in the Twingate Admin Console
- qemu-img.exe is resolved automatically from the latest fdcastel GitHub release (with a Cloudbase v2.3.0 fallback); no manual download needed.

### Parameters

| Parameter | Required | Default | Description |
|---|---|---|---|
| `-Action` | Yes | — | `Deploy`, `Remove`, `UpdateConnector`, `UpdateOS`, `List`, or `FixVM` |
| `-TwingateNetwork` | Most actions | prompted | Your Twingate network — the part of your Admin Console URL before `.twingate.com`. For `acme.twingate.com` use `acme`; for a shard-based URL like `acme.us1.twingate.com` use `acme.us1`. Copy it from the console rather than assuming a single label. |
| `-ApiToken` | Most actions | prompted | API token. Accepts plain string or SecureString. |
| `-RemoteNetwork` | Deploy, Remove | prompted | Remote Network display name from the Admin Console |
| `-ConnectorCount` | Deploy only | `2` | Number of connectors (and VMs) to create |
| `-VMPath` | No | `C:\TwingateConnectors` | Root directory for VM files, cached images, and tools |
| `-VMCpu` | Deploy only | `1` | vCPUs per VM |
| `-VMMemory` | Deploy only | `2147483648` (2 GB) | RAM per VM in bytes |
| `-VSwitch` | No | auto-detect | Hyper-V external vSwitch name |
| `-VMName` | FixVM (required); Remove (optional) | prompted for FixVM | Target a single VM by name, e.g. `TG-Connector-NY-1` |

`List` requires no API parameters. `UpdateConnector` and `UpdateOS` require `-TwingateNetwork` and `-ApiToken` but not `-RemoteNetwork` — they discover VMs by name pattern (`TG-Connector-*`).

### Actions

**Deploy** — Creates connectors via the Twingate API, provisions Ubuntu 24.04 Gen2 VMs using cloud-init, and waits for all connectors to report `ALIVE`. On first run, downloads the Ubuntu cloud image (~600 MB) and qemu-img.exe to `VMPath\images` and `VMPath\tools` respectively. These are cached and reused on subsequent runs.

**Remove** — Stops and deletes all VMs matching `TG-Connector-<RemoteNetwork>-*`, removes their disk files, and deletes the corresponding connector records from the Twingate API. Pass `-VMName <name>` to remove just one VM (its connector record and files); in that mode `-RemoteNetwork` is not required. Without `-VMName`, Remove targets all `TG-Connector-<RemoteNetwork>-*` VMs as before. Fail-safe: if the API delete fails, local VM cleanup still proceeds with a warning.

**UpdateConnector** — SSHs into each running VM sequentially and runs `apt-get install twingate-connector` to upgrade to the latest connector package. Verifies the connector reports `ALIVE` before moving to the next VM.

**UpdateOS** — SSHs into each running VM sequentially and runs `apt-get upgrade` to apply all OS updates. Verifies the connector reports `ALIVE` before moving to the next VM.

**List** — Displays all `TG-Connector-*` VMs with their Hyper-V state, IP address, uptime, and connector ID. No API call required.

**FixVM** — Repairs a single connector VM by name (`-VMName`). Checks whether the connector is already `ALIVE` (no-op), installed but stopped (starts it), or missing entirely (creates a **net-new** connector, runs the bootstrap over SSH, and repoints the VM). When it reprovisions, the VM's previous connector record is left in the Twingate Admin Console and flagged in an "ACTION REQUIRED" notice at the end of the run so you can review/remove it manually.

### Usage Examples

```powershell
# Deploy 2 connectors (default) into the "Office" Remote Network
.\Deploy-TwingateConnector.ps1 -Action Deploy -TwingateNetwork "acme" -RemoteNetwork "Office"

# Deploy 4 connectors with more RAM, storing files on D:\
.\Deploy-TwingateConnector.ps1 -Action Deploy -TwingateNetwork "acme" -RemoteNetwork "Office" `
    -ConnectorCount 4 -VMPath D:\VMs -VMMemory 4GB

# List all connector VMs
.\Deploy-TwingateConnector.ps1 -Action List

# Remove all connectors in a Remote Network
.\Deploy-TwingateConnector.ps1 -Action Remove -TwingateNetwork "acme" -RemoteNetwork "Office"

# Update the twingate-connector package on all VMs
.\Deploy-TwingateConnector.ps1 -Action UpdateConnector -TwingateNetwork "acme"

# Update the OS on all VMs
.\Deploy-TwingateConnector.ps1 -Action UpdateOS -TwingateNetwork "acme"

# Pass the API token as a SecureString (avoids plain text in shell history)
.\Deploy-TwingateConnector.ps1 -Action Deploy -TwingateNetwork "acme" -RemoteNetwork "Office" `
    -ApiToken (ConvertTo-SecureString 'your-token' -AsPlainText -Force)

# Repair a single connector VM
.\Deploy-TwingateConnector.ps1 -Action FixVM -TwingateNetwork "acme" -VMName "TG-Connector-NY-1"

# Remove just one VM (RemoteNetwork not required)
.\Deploy-TwingateConnector.ps1 -Action Remove -TwingateNetwork "acme" -VMName "TG-Connector-NY-1"
```

### What the script creates

For each connector, the following is created under `VMPath\TG-Connector-<RemoteNetwork>-<N>\`:

| File | Description |
|---|---|
| `disk.vhdx` | VM OS disk (copy of Ubuntu base image, expanded to 20 GB) |
| `cloud-init.iso` | NoCloud datasource ISO with cloud-init configuration |
| `ssh_key` / `ssh_key.pub` | Per-VM ED25519 SSH keypair |
| `ssh_user.txt` | VM admin username (used by Update actions) |

The Ubuntu `ubuntu` default user is disabled. A randomly generated admin user (`tgadm` + 4 random characters) is created with a 24-character random password. The script prints the credentials to the console at deploy time — save them if you need console/emergency access.

### VM naming convention

VMs are named `TG-Connector-<RemoteNetwork>-<N>`, e.g.:
- `TG-Connector-Office-1`
- `TG-Connector-Office-2`

Discovery in Remove and Update actions uses the `TG-Connector-*` name pattern. Connector IDs are stored in the VM's Notes field.

### Cleanup / reset

Use `Reset-TwingateConnectorEnvironment.ps1` to remove all `TG-Connector-*` VMs, their files, and the `TwingateExternalSwitch` vSwitch in one shot — useful after a failed or interrupted deployment. Cached downloads (`images\` and `tools\`) are intentionally left intact.

```powershell
# Preview what will be removed
.\Reset-TwingateConnectorEnvironment.ps1

# Remove everything without prompting
.\Reset-TwingateConnectorEnvironment.ps1 -Force
```

---

## Tests

The [`tests/`](tests/) directory contains checks intended to be run **locally** if you are modifying `Deploy-TwingateConnector.ps1`. They are not wired into CI and are not required to use the scripts.

| Test | Runs with | Covers |
|---|---|---|
| [`tests/Deploy-TwingateConnector.Tests.ps1`](tests/Deploy-TwingateConnector.Tests.ps1) | [Pester](https://pester.dev) 5+ (`Invoke-Pester ./tests`) | Helper functions — status output, cloud-init user-data generation |
| [`tests/test-bootstrap-retry.sh`](tests/test-bootstrap-retry.sh) | Bash (WSL, Git Bash, or Linux) | Connector bootstrap retry loop; stubs `curl`/`dpkg`/`systemctl` so no network or root is needed |

The Pester file dot-sources only the script's *Helper Functions* region, so it does not execute the param block or any action handler. The bash test takes the path to an emitted `bootstrap.sh` as its argument.

---

## Issues and contributions

Because this is an unsupported example project, there is no SLA on responses — but bug reports and feedback are genuinely welcome and help gauge interest in Hyper-V as a Connector platform.

- **Found a bug?** Open an issue with your Windows Server and PowerShell versions, the `-Action` you ran, and the console output (redact your API token, network slug, and any connector tokens).
- **Have an improvement?** Pull requests are welcome. Please run the tests above against `Deploy-TwingateConnector.ps1` before submitting.
- **Need supported help with Twingate itself?** Contact Twingate through official support channels rather than this repository.

---

## License

Apache 2.0 — see [LICENSE](LICENSE).
