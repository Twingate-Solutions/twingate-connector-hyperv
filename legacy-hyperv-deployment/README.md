# Legacy Hyper-V Connector Installer

> ⚠️ **Example project — provided as-is, with no support or warranty.** These scripts are published as a reference example to build from, not a supported product. Nothing here is guaranteed and no support is attached to it. They were developed with help from an LLM-based coding assistant. Review the code and test it yourself before using it in any critical or production environment. Your use is governed by the Apache License 2.0, including its "AS IS", no-warranty (Section 7), and limitation-of-liability (Section 8) terms.

> 🕒 **Deprecated.** This is the original deployment approach, kept for the narrow cases below. For new deployments use [`Deploy-TwingateConnector.ps1`](../Deploy-TwingateConnector.ps1) in the repository root — see the [main README](../README.md).

## Overview

`hyperv-prebuilt-image-connector-installer.ps1` downloads a pre-built Ubuntu 22.04 VM archive from this repository's [Releases](https://github.com/Twingate-Solutions/twingate-connector-hyperv/releases) page, extracts it, and imports it as a Hyper-V VM. It then connects over SSH and installs the Twingate Connector.

Connector tokens must be generated manually in the Twingate Admin Console and pasted into the script before running.

## When to use

Use this script only if:

- You are on a system where the newer script's automatic image conversion doesn't work
- You specifically need Ubuntu 22.04

For all other cases, use [`Deploy-TwingateConnector.ps1`](../Deploy-TwingateConnector.ps1), which is API-driven, deploys multiple connectors, and supports full lifecycle management.

## Prerequisites

- Windows Server or Windows Desktop with Hyper-V available (the script can install the role and prompt for a reboot)
- PowerShell 5.1+ running **as Administrator**
- Internet access
- The [Posh-SSH](https://www.powershellgallery.com/packages/Posh-SSH) module — installed automatically if missing
- An external Hyper-V virtual switch, or a NIC named `Ethernet` so the script can create one
- Free disk space for both the ~1.8 GB archive and the ~5.6 GB extracted VHDX

## Setup

1. In the Twingate Admin Console, create a Connector and copy the **Access Token** and **Refresh Token**.
2. Open `hyperv-prebuilt-image-connector-installer.ps1` and set the three variables at the top:

   ```powershell
   $networkName    = "companyname"   # your Twingate network slug
   $accessToken    = "eyJhbG..."     # access token from the Admin Console
   $refreshToken   = "80zwhs..."     # refresh token from the Admin Console
   ```

3. Run the script as Administrator.

## What it does

1. Detects Windows Server vs Desktop and installs Hyper-V if needed (reboot required).
2. Installs the Posh-SSH module if not present.
3. Downloads the VM archive to `C:\windows\temp\` (skipped if already present).
4. Extracts it to `C:\twingate-connector-hyperv`.
5. Reuses an existing external vSwitch, or creates `TwingateExternalSwitch` bound to the `Ethernet` adapter.
6. Imports the VM as `Ubuntu_Twingate_Connector-22_04`, attaches it to the switch, and starts it.
7. Waits 120 seconds, then connects over SSH to hostname `twingate-connector` and runs the Twingate Connector setup script.

## Limitations and known issues

- **Tokens are hardcoded in the script.** Rotate them if the script is shared or committed to source control.
- **Single connector only.** Run the script again with different tokens for each additional connector.
- **No lifecycle management** — no Remove, Update, or List actions.
- **Ubuntu 22.04 LTS**, versus 24.04 in the newer script.
- **The published image was built in August 2024**, so it carries a significant OS patch gap. Run `sudo apt update && sudo apt upgrade` in the guest after import.
- **Default guest credentials.** The image ships with the user `twingate` and password `twingate`, which the script uses to connect over SSH. Change this immediately after deployment.
- **The script runs `chmod -R 0777 /etc/twingate`**, making connector configuration and credentials world-readable and world-writable. Tighten these permissions after deployment.
- **Boot wait is a fixed 120-second sleep**, not a readiness check. On slower hosts the SSH step may run before the guest is ready.
- **The guest is reached by hostname** (`twingate-connector`) over DHCP, which requires working name resolution on your network.
- **The Hyper-V detection check is known to be imperfect** on Windows Desktop — see the `TO DO` comment in the script.

## License

Apache 2.0 — see [LICENSE](../LICENSE).
