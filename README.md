# OCI Always Free Arm (A1.Flex) VM Creator with Feishu Notifications

[![Try to Create OCI VM](https://github.com/znzn007007/oci-free-arm-instance/actions/workflows/create-vm.yml/badge.svg)](https://github.com/znzn007007/oci-free-arm-instance/actions/workflows/create-vm.yml)

This fork runs a GitHub Actions workflow that repeatedly tries to create an Oracle Cloud Infrastructure (OCI) Always Free `VM.Standard.A1.Flex` Arm instance.

It is meant for the common OCI free-tier situation where A1.Flex capacity is temporarily unavailable and the OCI API returns:

```text
Out of host capacity.
```

The workflow keeps retrying on a schedule and sends Feishu/Lark notifications only when something needs attention.

## Current Behavior

* Runs on GitHub-hosted `ubuntu-latest`; no self-hosted runner is required.
* Runs every 5 minutes.
* Can also be started manually from the GitHub Actions page.
* Uses repository secrets for OCI credentials, region, image, subnet, availability domain, SSH key, and Feishu webhook.
* Tries to create one `VM.Standard.A1.Flex` instance.
* Default instance settings are `2 OCPU`, `12 GB RAM`, `200 GB boot volume`, and display name `coolify-vm`.
* Does not send Feishu messages for expected `Out of host capacity` retries.
* Sends Feishu messages for successful provisioning and non-capacity errors.
* Supports Feishu/Lark custom bot signature verification through `FEISHU_WEBHOOK_SECRET`.

## Required Repository Secrets

Go to:

```text
Settings -> Secrets and variables -> Actions -> Secrets -> New repository secret
```

Add these repository secrets:

| Secret | Value |
| --- | --- |
| `OCI_COMPARTMENT_ID` | The compartment OCID to create the VM in. For simple Always Free setups, the tenancy OCID often works here. |
| `OCI_SUBNET_ID` | The subnet OCID where the VM should be created. |
| `AD_NAME` | Availability domain name, for example `xxxx:US-PHOENIX-1-AD-1`. |
| `IMAGE_ID` | Arm/aarch64 image OCID for your region. The image must support `VM.Standard.A1.Flex`. |
| `OCI_CLI_REGION` | OCI region identifier, for example `us-phoenix-1`. |
| `OCI_CLI_USER` | User OCID, starting with `ocid1.user...`. |
| `OCI_CLI_TENANCY` | Tenancy OCID, starting with `ocid1.tenancy...`. |
| `OCI_CLI_FINGERPRINT` | OCI API key fingerprint. |
| `OCI_CLI_KEY_CONTENT` | Full private key content, from `-----BEGIN PRIVATE KEY-----` to `-----END PRIVATE KEY-----`. |
| `SSH_PUBLIC_KEY` | Full SSH public key line, starting with `ssh-rsa` or `ssh-ed25519`. |
| `FEISHU_WEBHOOK_URL` | Feishu/Lark custom bot webhook URL. |

Optional:

| Secret | Value |
| --- | --- |
| `FEISHU_WEBHOOK_SECRET` | Feishu/Lark custom bot signing secret, only needed if signature verification is enabled. |

## OCI Values

### Tenancy, User, and Region

In the OCI Console:

1. Open the profile menu in the top-right corner.
2. Open tenancy details and copy the tenancy OCID for `OCI_CLI_TENANCY`.
3. Open user settings and copy the user OCID for `OCI_CLI_USER`.
4. Copy the region identifier for `OCI_CLI_REGION`, such as `us-phoenix-1`.

`OCI_CLI_TENANCY` must look like:

```text
ocid1.tenancy.oc1..xxxxxxxx
```

Do not paste `tenancy=...`, a user OCID, a compartment OCID, or the tenancy display name.

### API Key

In OCI user settings:

1. Open **API Keys**.
2. Add or generate an API key.
3. Copy the fingerprint into `OCI_CLI_FINGERPRINT`.
4. Copy the full private key file content into `OCI_CLI_KEY_CONTENT`.

### Subnet and Availability Domain

For `OCI_SUBNET_ID`, open:

```text
Networking -> Virtual cloud networks -> your VCN -> Subnets -> your public subnet
```

Copy the subnet OCID.

For `AD_NAME`, open the create-instance page and copy the exact availability domain name shown in placement settings.

### Image ID

`IMAGE_ID` must be an Arm/aarch64 image for the same region as `OCI_CLI_REGION`.

For A1.Flex, do not use the normal x86 Ubuntu image. Use an image whose name includes `aarch64`, such as:

```text
Canonical-Ubuntu-24.04-aarch64-...
```

Oracle image OCIDs are listed here:

```text
https://docs.oracle.com/en-us/iaas/images/
```

### SSH Public Key

On your Mac, run:

```bash
cat ~/.ssh/id_rsa.pub
```

If that file does not exist, try:

```bash
cat ~/.ssh/id_ed25519.pub
```

Copy the full output line into `SSH_PUBLIC_KEY`. It should start with `ssh-rsa` or `ssh-ed25519`, not just the trailing machine name.

Example shape:

```text
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... nemo@MacBook
```

## Feishu/Lark Bot

In Feishu/Lark:

1. Open the group that should receive notifications.
2. Add a custom bot.
3. Copy the webhook URL into `FEISHU_WEBHOOK_URL`.
4. If signature verification is enabled, copy the signing secret into `FEISHU_WEBHOOK_SECRET`.

The workflow automatically adds `timestamp` and `sign` when `FEISHU_WEBHOOK_SECRET` is present.

## Run the Workflow

Open:

```text
Actions -> Try to Create OCI VM -> Run workflow
```

Choose `main`, then start the run.

After that, the schedule will retry every 5 minutes.

## Notifications

The Feishu notification policy is intentionally quiet:

| Result | Feishu notification |
| --- | --- |
| `Out of host capacity` | No notification |
| Successful provisioning / `PROVISIONING` | Notification sent |
| Credential, OCID, subnet, image, SSH key, or other non-capacity errors | Notification sent |

You can still inspect every scheduled attempt in GitHub Actions logs.

## What to Do on Success

When Feishu reports success, or the GitHub Actions log contains:

```text
"lifecycle-state": "PROVISIONING"
```

disable the workflow immediately:

```text
Actions -> Try to Create OCI VM -> ... -> Disable workflow
```

If you leave it enabled, it will keep trying every 5 minutes and may attempt to create another VM.

## Troubleshooting

### `tenancy | malformed | this must be an OCID`

`OCI_CLI_TENANCY` is not a valid tenancy OCID. It must start with:

```text
ocid1.tenancy...
```

### `Invalid ssh public key type`

`SSH_PUBLIC_KEY` is incomplete. Copy the full public key line, starting with `ssh-rsa` or `ssh-ed25519`.

### `Shape VM.Standard.A1.Flex is not valid for image`

`IMAGE_ID` is probably an x86 image. Replace it with an Arm/aarch64 image OCID from the same region.

### `Out of host capacity`

This is the expected retry state. The configuration is working; OCI does not currently have A1.Flex capacity in that availability domain.

### `There are no runners configured`

This is normal. The workflow uses GitHub-hosted `ubuntu-latest`, so you do not need to create a self-hosted runner.

## Changing VM Size

The VM size is currently hardcoded in `.github/workflows/create-vm.yml`:

```yaml
--shape-config '{"ocpus":2,"memoryInGBs":12}' \
--display-name "coolify-vm" \
--boot-volume-size-in-gbs 200
```

For a smaller VM, edit those values in the workflow file.

## License

This repository is available under the [MIT License](LICENSE).
