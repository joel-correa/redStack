# redStack: A Boot-to-Breach Lab Environment for Red Team Operators

![redStack Banner](assets/redStack-banner.png)

> [!NOTE]
> redStack is now feature complete and supports both public internet deployments and closed environments (HTB/VL/PG) that use OpenVPN. This is actively being tested and debugged, so please reach out with any issues or concerns.

> [!IMPORTANT]
> redStack is not a tutorial on how to use C2 frameworks. It is an environment that removes the infrastructure hurdle so you can focus on learning. The lab gives you a fully configured, production-style red team setup out of the box (Boot-to-Breach). **This lab is strictly for authorized training and lab environments only (HTB, VL, PG, self-hosted cyber ranges, personal lab VMs, etc.). It is not intended for use in real-world engagements or against targets you do not own and have explicit written permission to test.**

> [!CAUTION]
> **AWS TOS: Use at your own risk**
>
> Hosting C2 infrastructure on AWS may raise concerns under the [AWS Acceptable Use Policy](https://aws.amazon.com/aup/). Before deploying, review the AUP and submit the [AWS Penetration Testing / Simulated Events request form](https://aws.amazon.com/security/penetration-testing/). This is the appropriate channel for notifying AWS that you are running security tooling on their infrastructure.
>
> As long as you are using redStack exclusively for personal lab work and authorized training platforms (HTB, VL, PG, self-hosted cyber ranges, etc.), you are generally in the clear. A quick conversation with AWS customer support can confirm this and give you peace of mind specific to your account and usage pattern. To be safe, consider running redStack from a dedicated, single-purpose throwaway AWS account used solely for this lab. That isolates billing alerts and removes any risk to other workloads or account standing.

---

## Quick Start

**What you deploy:** 6 EC2 instances (3 C2 servers + Apache redirector + Guacamole jumpbox + Windows operator workstation) across 2 peered VPCs in a single AWS account.

**Time to ready:** ~10 to 15 min `terraform apply` + ~5 to 10 min cloud-init, plus a one-time SSL cert and Havoc build on first deploy.

**Cost:** ~$5 to $9/mo for 5 to 10 hrs/wk of study with `terraform destroy` between sessions (recommended). ~$26 to $30/mo if you stop but do not destroy between sessions. ~$172/mo if left running 24/7. Full breakdown in [Cost Management](#cost-management).

**Pick a mode** (full decision tree in [Deployment Modes](#deployment-modes)):

- **Open environment** (default): public domain + Let's Encrypt HTTPS. Use for general study, payload testing, AV evasion practice. Requires a domain you own.
- **Closed environment**: HTB / VulnLab / Proving Grounds Pro Labs over OpenVPN. No public DNS, self-signed cert. Skip the domain steps and follow [Part 8](#part-8-external-target-environments-htbvlpg).

**You will need:**

- AWS account (a dedicated throwaway is strongly recommended) with EC2 quota for at least 6 instances and 2 VPCs
- Terraform >= 1.0 and AWS CLI
- An RSA SSH key pair created in AWS (Step 0.4)
- A domain name (open environment only)
- ~$30/mo budget

**The 6-step path:** Part 0 (prerequisites) > Part 1 (`terraform apply`) > Part 2 (verify access) > Part 3 (SSL + redirector) > Parts 4 to 6 (Mythic / Sliver / Havoc) > Cleanup with `terraform destroy`.

---

## Table of Contents

- [Quick Start](#quick-start)
- [Architecture Overview](#architecture-overview)
- [Lab Inventory](#lab-inventory)
- [SSH Access](#ssh-access)
- [Deployment Modes](#deployment-modes)
- [Timing Expectations](#timing-expectations)
- [Part 0: Pre-Deployment Checklist](#part-0-pre-deployment-checklist)
  - [Prerequisites](#prerequisites)
  - [Step 0.1: Clone Repository & Install Tools](#step-01-clone-repository--install-tools)
  - [Step 0.2: AWS IAM Permissions](#step-02-aws-iam-permissions)
  - [Step 0.3: Verification Commands](#step-03-verification-commands)
  - [Step 0.4: Create AWS SSH Key Pair (Required)](#step-04-create-aws-ssh-key-pair-required)
- [Part 1: Initial Deployment](#part-1-initial-deployment)
  - [Step 1.1: Configure Terraform Variables](#step-11-configure-terraform-variables)
  - [Step 1.2: Initialize Terraform](#step-12-initialize-terraform)
  - [Step 1.3: Review Deployment Plan](#step-13-review-deployment-plan)
  - [Step 1.4: Deploy Infrastructure](#step-14-deploy-infrastructure)
  - [Step 1.5: Review Deployment Information](#step-15-review-deployment-information)
  - [Step 1.6: Point Domain to Redirector](#step-16-point-domain-to-redirector)
- [Part 2: Verification](#part-2-verification)
  - [Step 2.1: Access Guacamole Portal](#step-21-access-guacamole-portal)
  - [Step 2.2: Access Windows Workstation](#step-22-access-windows-workstation)
  - [Step 2.3: Verify Internal Connectivity](#step-23-verify-internal-connectivity)
- [Post-Deployment Actions](#post-deployment-actions)
  - [Cleanup (When Done)](#cleanup-when-done)
  - [Cost Management](#cost-management)
- [Success Criteria](#success-criteria)
- [Operating the Lab (Wiki)](#operating-the-lab-wiki)

---

## Architecture Overview

```text
+----------------------------------------------------------------------+
|                    redStack Network Architecture                     |
+----------------------------------------------------------------------+

                          [ Operator ]
                       Browser / MobaXterm
                               |
                   HTTPS :443  |  SSH :22
                               |
+------------------------------+------------------------------+
|               TeamServer VPC (172.31.0.0/16)                |
|   +-----------------------------------------------------+   |
|   | guacamole               Elastic IP: <Public IP>     |   |
|   | 172.31.x.x                                          |   |
|   +--+----+---------+-----------------------------------+   |
|      |    |         |         Guacamole-managed sessions    |
|     SSH  SSH     SSH/RDP                                    |
|      |    |         |                                       |
|      +----+---------+--------+----------+                   |
|      |    |         |        |          |                   |
|      v    v         v        v          v                   |
| +------++------++------+  +------------+ +------+            |
| |mythic||sliver||havoc |  |win srv22   | | kali |            |
| +------++------++------+  +------------+ +------+            |
|        ( no public IPs - internal only )                    |
+------------------------------+------------------------------+       
                               |                              
           VPC Peering: 172.31.0.0/16 <-> 10.60.0.0/16        
           - C2 callbacks: Apache proxy -> teamservers
                               |                               
+------------------------------+------------------------------+
|                Redirector VPC (10.60.0.0/16)                |
|   +-----------------------------------------------------+   |
|   | redirector              Elastic IP: <Public IP>     |   |
|   | 10.60.x.x                                           |   |
|   | Apache :80/:443 (X-Request-ID + URI validation)     |   |
|   | Decoy page served to unvalidated requests           |   |
|   +-----------------------------------------------------+   |
+------------------------------+------------------------------+
                               ^
                               |
                    public internet / cloud DNS
                               |
                               v
          [ Public Internet Accessible Target Environments ]

Public Internet Environment (C2 Callback Flow):
  [target / implant] --HTTPS/HTTP--> public internet / cloud DNS
  --> redirector Elastic IP --> Apache (X-Request-ID + URI validation)
  --> VPC peering --> mythic / sliver / havoc (172.31.x.x)
```

> [!NOTE]
>
> - All C2 servers have no public IPs. Reachable only through the redirector via VPC peering
> - The redirector runs in its own isolated VPC, simulating an external provider
> - Every lab machine has `/etc/hosts` entries so all hostnames resolve across the environment
> - Requests without a valid `X-Request-ID` header receive a decoy CloudEdge CDN maintenance page
> - Only requests with a matching URI prefix and the correct header token are proxied to the correct C2 server
> - `redirect.rules` blocks AV vendors and TOR exits (403)
> - Run `terraform output network_architecture` to see the diagram populated with your actual IPs

---

## Lab Inventory

Seven instances are deployed by default. Only the Guacamole portal and the Apache redirector hold public Elastic IPs. Everything else lives inside private VPCs and is reached through Guacamole.

| Hostname | Role | Public IP | Default access | Credentials source |
| --- | --- | --- | --- | --- |
| `guac` | Guacamole portal (web SSH/RDP/VNC) | Yes | `https://<guac-eip>/guacamole` | `terraform output deployment_info` |
| `redirector` | Apache reverse proxy / C2 frontend | EIP exposes 80/443 only | Guacamole > Apache Redirector SSH (or `ssh -J` via guac) | `terraform output deployment_info` |
| `mythic` | Mythic C2 server | No | Guacamole > Mythic SSH (or `ssh -J` via guac) | `terraform output deployment_info` |
| `sliver` | Sliver C2 server | No | Guacamole > Sliver SSH (or `ssh -J` via guac) | `terraform output deployment_info` |
| `havoc` | Havoc C2 server + Havoc desktop (VNC) | No | Guacamole > Havoc SSH or VNC | `terraform output deployment_info` |
| `WIN-OPERATOR` | Windows operator workstation | No | Guacamole > Windows Operator (RDP) | AWS-decrypted, shown in `terraform output deployment_info` |
| `kali` | Kali Linux operator (AD enum + attack toolset) | No | Guacamole > Kali Operator SSH (and XRDP if `kali_deployment_mode = "gui"`) | `terraform output deployment_info` |

**One command for everything:** `terraform output deployment_info` prints all IPs, the auto-generated lab password, the Windows Administrator password, and the C2 header token. Save it once after deploy.

---

## SSH Access

Two facts make the access pattern simple:

1. The same `rs-rsa-key.pem` (the AWS key pair) is authorized on every Linux instance.
2. sshd on each Linux host is configured `PasswordAuthentication no` for public sources, but flips to `yes` for source addresses inside `172.16.0.0/12` or `10.0.0.0/8` (the lab's private space). See [redirector_userdata.sh:39-48](setup_scripts/redirector_userdata.sh#L39-L48) for the canonical config.

So from your host workstation (a public source), you must use the key. From a shell already inside the lab (a private source), the auto-generated lab password works too.

> [!IMPORTANT]
> Only Guacamole's public EIP exposes port 22. The redirector's EIP exposes 80 and 443 only. C2 servers and the Windows workstation have no public IP. Every SSH session into the lab must enter through Guacamole.

**Easiest: Guacamole web portal.** The "Apache Redirector (SSH)", "Mythic C2 Server (SSH)", "Sliver C2 Server (SSH)", and "Havoc C2 Server (SSH)" tiles are pre-configured with the lab password. No manual auth.

**SSH directly to Guacamole (the bastion):**

```bash
ssh -i rs-rsa-key.pem admin@<GUAC_PUBLIC_IP>
```

**Jump from your host to any internal box in one command** (hostname is resolved on guac via `/etc/hosts`):

```bash
ssh -i rs-rsa-key.pem -J admin@<GUAC_PUBLIC_IP> admin@redirector
ssh -i rs-rsa-key.pem -J admin@<GUAC_PUBLIC_IP> admin@mythic
ssh -i rs-rsa-key.pem -J admin@<GUAC_PUBLIC_IP> admin@sliver
ssh -i rs-rsa-key.pem -J admin@<GUAC_PUBLIC_IP> admin@havoc
```

The same `-i` key satisfies both hops because it is authorized on every box. No password prompts.

**From a shell already on guac, hop directly to any internal host:**

```bash
admin@guac:~$ ssh admin@redirector    # private source, password or key both work
admin@guac:~$ ssh admin@mythic
admin@guac:~$ ssh admin@sliver
admin@guac:~$ ssh admin@havoc
```

`/etc/hosts` on every Linux host maps every other lab hostname to its private IP, so `admin@<hostname>` resolves correctly. Use `<HOST_PRIVATE_IP>` instead if you prefer.

---

## Deployment Modes

redStack runs in one of two modes. Pick before you fill out `terraform.tfvars`. The choice changes only a handful of variables and a couple of post-deploy steps.

| | **Open environment** (default) | **Closed environment** (HTB / VL / PG) |
| --- | --- | --- |
| **Use when** | General study, payload testing, AV evasion practice over the public internet | Pro Labs reachable only over OpenVPN (HackTheBox, VulnLab, Proving Grounds) |
| **DNS** | You own a public domain and create A records to the redirector | None. Redirector is reached by its public Elastic IP |
| **TLS** | Let's Encrypt cert via Certbot (manual one-time step) | Self-signed cert with the public IP as Subject Alternative Name (auto-generated at deploy) |
| **Scanner blocking** | `redirect.rules` enabled (blocks AV vendors, TOR exits) | Disabled (not needed in isolated lab networks) |
| **VPN tunnel** | None | OpenVPN client on redirector + WireGuard tunnel from C2 VPC > redirector |
| **Key tfvars** | `redirector_domain = "yourdomain.tld"` | `redirector_domain = ""`, `enable_external_vpn = true`, `enable_redirector_htaccess_filtering = false` |
| **Path through this guide** | Parts 0 to 7 in order | Parts 0 to 7, but skip Steps 1.6 and 3.1; then follow [Part 8](#part-8-external-target-environments-htbvlpg) |

> [!NOTE]
> Both modes use the same network architecture, the same C2 stack, and the same Guacamole portal. Only the front door changes.

---

## Timing Expectations

Set expectations before you start. Most timing surprises in redStack come from the Windows boot and the Havoc build.

| Phase | Duration | Notes |
| --- | --- | --- |
| `terraform apply` | ~10 to 15 min | 50+ AWS resources |
| Cloud-init on Linux hosts | ~5 to 10 min | Mythic Docker pulls, Apache config, redirect.rules download |
| Windows initialization | up to ~15 min | Slowest component. RDP becomes available last |
| Certbot SSL issuance (open mode only) | ~30 sec | After DNS propagates |
| Havoc build (one-time, first deploy only) | ~15 to 25 min | Compiles teamserver from source on `havoc` |
| Mythic agent build | ~30 to 60 sec | Per agent |
| First C2 callback after agent execution | ~10 sec | Depending on `callback_interval` |
| `terraform destroy` | ~5 min | Releases EIPs, terminates instances, removes VPCs |

---

## Part 0: Pre-Deployment Checklist

### Prerequisites

- [ ] AWS account with IAM credentials
- [ ] AWS CLI installed and configured
- [ ] Terraform >= 1.0 installed
- [ ] Your public IP obtained
- [ ] Repository cloned (see Step 0.1)
- [ ] SSH key pair created in AWS EC2 (see Step 0.4 below)

### Step 0.1: Clone Repository & Install Tools

**Clone the repository:**

```bash
git clone https://github.com/BaddKharma/redStack.git
cd redStack
```

> [!NOTE]
> All subsequent commands should be run from inside the `redStack/` directory. This ensures the SSH key pair created in Step 0.4 lands in the right place.

**Install AWS CLI:**

| Platform | Command |
| -------- | ------- |
| macOS | `brew install awscli` |
| Linux (Ubuntu/Debian) | `sudo apt install awscli` |
| Linux (any) | `curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip" && unzip awscliv2.zip && sudo ./aws/install` |
| Windows | Download and run the MSI installer from <https://aws.amazon.com/cli/> |

**Install Terraform:**

| Platform | Command |
| -------- | ------- |
| macOS | `brew install terraform` |
| Linux (Ubuntu/Debian) | See <https://developer.hashicorp.com/terraform/install> |
| Windows | `choco install terraform` or download from <https://developer.hashicorp.com/terraform/install> |

**Checkpoint:** ✅ Repository cloned, AWS CLI and Terraform installed

### Step 0.2: AWS IAM Permissions

redStack provisions EC2, VPC, security group, Elastic IP, network interface, key pair, and IAM resources. Your AWS credentials need sufficient permissions to create and destroy all of these.

**For both options:** Go to **IAM Console** > **Users** > **Create user**, set a username (e.g., `redS-operator`), then under **Security credentials** create an access key and save the Access Key ID and Secret Access Key.

**Option A: AdministratorAccess (recommended. Use this unless you have a specific reason not to)**

This is the right choice for the vast majority of redStack users.

If you followed the earlier recommendation and created a dedicated AWS account solely for this lab, `AdministratorAccess` is the practical default. There are no other workloads, billing resources, or sensitive data in the account to protect. On an empty account, admin access carries the same real-world risk as a scoped policy. If the credentials are compromised, the attacker can only touch the lab infrastructure you already plan to tear down.

Least privilege adds meaningful protection when credentials could expose things beyond this lab. On a dedicated account, there is nothing else to expose. Use Option A and save Option B for when it actually buys you something.

<details>
<summary>How to create the IAM user and attach AdministratorAccess (click to expand)</summary>

```yaml
Step 1: IAM Console > Users > Create user
Step 2: Username - redS-operator
Step 3: Permissions > Attach policies directly > search "AdministratorAccess"
Step 4: Check AdministratorAccess > Next > Create user
Step 5: Open the new user > Security credentials > Create access key
Step 6: Select - Command Line Interface (CLI) > acknowledge > Next
Step 7: Copy the Access Key ID and Secret Access Key (the secret is shown only once)
```

</details>

**Option B: Least Privilege (only if you are deploying into a shared or production account)**

Use this option if the AWS account running redStack also contains other workloads, active resources, or anything you cannot afford to lose or expose. In that context, scoping the credentials to only what redStack needs limits the blast radius if the access key is ever leaked or misused.

The policy grants `ec2:*` for all Terraform operations, `sts:GetCallerIdentity` so Terraform can verify credentials at init, and four read-only IAM actions scoped to your own user so you can inspect your own permissions when troubleshooting. Nothing outside of EC2 and self-inspection is granted.

If you are not sure which account type you have, go back and re-read the dedicated account recommendation at the top of this section. Setting up a separate account is a one-time five-minute task and removes the need for this option entirely.

<details>
<summary>How to create the IAM user and attach the least-privilege policy (click to expand)</summary>

```yaml
Step 1:  IAM Console > Users > Create user
Step 2:  Username - redS-operator
Step 3:  Permissions > Attach policies directly > Create policy
Step 4:  Select the JSON tab and paste the policy shown below
Step 5:  Name the policy - redStack-least-privilege > Create policy
Step 6:  Back on the user creation screen, search for and attach redStack-least-privilege
Step 7:  Next > Create user
Step 8:  Open the new user > Security credentials > Create access key
Step 9:  Select - Command Line Interface (CLI) > acknowledge > Next
Step 10: Copy the Access Key ID and Secret Access Key (the secret is shown only once)
```

</details>

<!-- -->

**Minimum IAM Policy:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:*",
        "sts:GetCallerIdentity"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "iam:GetUser",
        "iam:GetUserPolicy",
        "iam:ListUserPolicies",
        "iam:ListAttachedUserPolicies"
      ],
      "Resource": "arn:aws:iam::*:user/${aws:username}"
    }
  ]
}
```

> [!NOTE]
> **Why these permissions:**
>
> - **`ec2:*`**: redStack is EC2-only infrastructure. Every resource Terraform creates and destroys (instances, VPCs, subnets, security groups, ENIs, EIPs, VPC peering) maps to an EC2 API call. No S3, RDS, Lambda, or other services are used.
> - **`sts:GetCallerIdentity`**: Terraform calls this at init to verify credentials and identify the account. Without it, `terraform init` fails before any resources are touched.
> - **`iam:GetUser`, `iam:GetUserPolicy`, `iam:ListUserPolicies`, `iam:ListAttachedUserPolicies`**: Read-only, self-scoped to `${aws:username}`. Lets you inspect your own permissions when debugging an access denied error. No IAM write access is granted and the scope prevents reading any other principal's policies.

**Configure AWS CLI:**

Running `aws configure` writes your credentials and preferences to `~/.aws/credentials` and `~/.aws/config`. This is how Terraform (and the AWS CLI) know which account to talk to and which region to deploy into. You only need to do this once per machine.

```bash
aws configure
```

<details>
<summary>What each prompt is asking for (click to expand)</summary>

- **AWS Access Key ID**: The access key you generated under **Security credentials** for your IAM user. Identifies which user is making requests.
- **AWS Secret Access Key**: The secret shown once at key creation time. Acts as the password paired with the Access Key ID. If you did not save it, delete the key and create a new one.
- **Default region name**: The AWS region where redStack will be deployed. Use `us-east-1` unless you have a specific reason to pick another. This must match the `aws_region` value in `terraform.tfvars`.
- **Default output format**: Controls how the AWS CLI formats responses. Use `json`. Terraform does not use this setting but it makes CLI output readable when troubleshooting.

</details>

**Checkpoint:** ✅ IAM user created and AWS CLI configured with the new credentials

### Step 0.3: Verification Commands

```bash
# Check AWS access
aws sts get-caller-identity

# Check Terraform
terraform --version

# Get your public IP
curl -s ifconfig.me
```

**Expected Results:**

- AWS CLI returns your account details (Account, Arn, UserId)
- Terraform version 1.0 or higher
- Public IP address displayed

**Checkpoint:** ✅ AWS CLI and Terraform working, public IP noted

### Step 0.4: Create AWS SSH Key Pair (Required)

**Terraform does NOT create the SSH key pair - you must create it manually first.**

<details>
<summary>Windows (PowerShell)</summary>

```powershell
aws ec2 create-key-pair --key-name rs-rsa-key --query 'KeyMaterial' --output text | Out-File -Encoding ascii rs-rsa-key.pem
icacls "rs-rsa-key.pem" /inheritance:r /grant:r "$($env:USERNAME):R"
```

</details>

<details>
<summary>Linux/Mac (bash)</summary>

```bash
aws ec2 create-key-pair --key-name rs-rsa-key --query 'KeyMaterial' --output text > ./rs-rsa-key.pem
chmod 400 ./rs-rsa-key.pem
```

</details>

**Verify key pair exists:**

```powershell
aws ec2 describe-key-pairs --key-names rs-rsa-key
```

> [!NOTE]
> You can also create the key pair in the AWS Console under EC2 > Key Pairs > Create key pair. Use RSA and .pem format. Download the file into your `redStack/` directory and fix permissions with the `icacls` command above.

**Checkpoint:** ✅ SSH key pair created and .pem file saved in project folder

### Step 0.5: Subscribe to the Kali Linux AMI (Required)

redStack provisions an official Kali Linux operator workstation alongside the C2 lab. Kali AMIs are publicly shared by the Kali project on AWS Marketplace at no charge, but AWS requires a one-time EULA acceptance per account before the first launch. **First-time `terraform apply` will fail with `OptInRequired` if this is skipped.**

**Steps:**

1. Visit <https://aws.amazon.com/marketplace/pp/prodview-fznsw3f7mq7to> while signed in to the same AWS account you configured the AWS CLI with.
2. Click **Continue to Subscribe**.
3. Click **Accept Terms**. The page will show "Subscribed" within ~30 seconds.

You do not need to launch anything from the Marketplace UI — Terraform will pick up the AMI automatically once the subscription is in place. The acceptance is permanent for the life of the account; you only do this once.

> [!NOTE]
> If you ran `terraform apply` and got an `OptInRequired` error, complete the subscription above and re-run `terraform apply`. No state cleanup needed — Terraform will retry the failed instance.

**Checkpoint:** ✅ Kali Linux AMI subscribed in AWS Marketplace

---

## Part 1: Initial Deployment

_Deploy all AWS infrastructure using Terraform: VPCs, security groups, EC2 instances (Mythic, Sliver, Havoc, Guacamole, Windows, Apache redirector)._

### Step 1.1: Configure Terraform Variables

```bash
cp terraform.tfvars.example terraform.tfvars
nano terraform.tfvars   # Linux/Mac
notepad terraform.tfvars  # Windows
```

**Required Changes (no defaults, must be set):**

```hcl
localPub_ip          = "YOUR_IP/32"           # Replace with your IP + /32
ssh_key_name         = "rs-rsa-key"           # Must match your AWS key pair name
ssh_private_key_path = "./rs-rsa-key.pem"     # Path to your .pem file (for Windows password decryption)
```

> [!NOTE]
> **Default deployment uses a public domain with htaccess filtering enabled.** This is the standard mode for open lab environments and is what the rest of this guide assumes. The closed environment option below is only for HTB/VL/PG Pro Labs and other isolated OpenVPN environments where no public DNS exists.
>
> **Open environment** (default: internet access, domain registered):
> Set `redirector_domain` to your domain. Complete Step 1.6 (DNS) and Step 3.1 (Certbot) to get a trusted TLS certificate. Agents call back using your domain over HTTPS. Scanner/AV blocking via `redirect.rules` is enabled by default.
>
> ```hcl
> redirector_domain = "c2.yourdomain.tld"
> ```
>
> **Closed environment** (HTB/VL/PG Pro Labs, OpenVPN-only, no public DNS):
> Leave `redirector_domain` empty and turn on the two external-VPN toggles below. The redirector uses its public Elastic IP as the server identity with a self-signed certificate. Scanner/AV blocking is disabled since it is not needed in isolated lab networks. Skip Step 1.6 and Step 3.1. See [Part 8](#part-8-external-target-environments-htbvlpg) for the full external-VPN deployment workflow.
>
> ```hcl
> # redirector_domain = ""                    # leave empty; redirector uses its public IP
> enable_external_vpn                  = true  # enables OpenVPN client + VPC routing
> enable_redirector_htaccess_filtering = false  # disables scanner/AV blocking (not needed in labs)
> ```

**Optional:** these have sensible defaults but affect callback URLs baked into payloads and VPN routing. Review before deploying:

```hcl
# Network: by default redStack creates a dedicated VPC (10.50.0.0/16) for the team server.
# This keeps the lab isolated and avoids conflicts with other AWS workloads in the same account.
# If you need to use an existing default VPC instead (e.g. if you hit the VPC limit), set:
#   use_default_vpc = true
# Leave vpc_cidr as-is unless the default range conflicts with something on your network.
use_default_vpc = false
vpc_cidr        = "10.50.0.0/16"

# Instance types: adjust for budget/performance
sliver_instance_type = "t3.small"
havoc_instance_type  = "t3.medium"

# C2 URI prefixes (CDN/cloud-style paths on the redirector)
# These are baked into payloads at deploy time. Customize before deploying.
mythic_uri_prefix = "/cdn/media/stream"
sliver_uri_prefix = "/cloud/storage/objects"
havoc_uri_prefix  = "/edge/cache/assets"

# --- External VPN routing for HTB / VulnLab / Proving Grounds Pro Labs ---
# Default redStack runs on the public internet (public domain + htaccess on).
# Toggle these only when internal lab machines need to reach Pro Lab targets
# through the redirector's OpenVPN tunnel. See Part 8 for the full flow.
enable_external_vpn                  = false  # true: OpenVPN client on redirector + WireGuard tunnel from C2 VPC
enable_redirector_htaccess_filtering = true   # false for HTB/VL/PG: scanner/AV blocking is not useful in isolated lab networks

# C2 header validation is always enabled. These override the defaults:
# c2_header_name  = "X-Request-ID"  # Header name (default: X-Request-ID)
# c2_header_value = ""              # Token value. Leave empty to auto-generate (recommended)

# Optional: custom tags applied to every AWS resource (instances, SGs, Elastic IPs, etc.)
# Useful for cost tracking and filtering resources in the AWS Console
tags = {
  Owner      = "Operator"
  CostCenter = "redStack"
  Purpose    = "Boot-to-Breach Training Environment"
}
```

> [!NOTE]
> **Header validation is always enabled.** `c2_header_name` and `c2_header_value` are optional overrides, not toggles. Leaving them out means the header name defaults to `X-Request-ID` and the token is auto-generated at deploy time. Retrieve the active token after deployment with `terraform output deployment_info`.

Passwords are auto-generated during deployment. A single random password is used for SSH and Guacamole admin access. The Windows Administrator password is generated by AWS and decrypted automatically using your SSH private key. Retrieve credentials after deployment with:

```bash
terraform output deployment_info
```

**Checkpoint:** ✅ File saved with your actual values

<details>
<summary><strong>Terraform Primer:</strong> new to Terraform? Click to expand.</summary>

If you are new to Terraform, here is a quick overview of the four commands used in this guide:

| Command | What it does |
| --- | --- |
| `terraform init` | Downloads provider plugins and initializes the working directory. Run once before anything else, or after adding new providers. |
| `terraform plan` | Dry run. Shows exactly what Terraform will create, change, or destroy. No changes are made. Useful for catching syntax or provisioning errors before a full `terraform apply`. |
| `terraform apply` | Provisions the infrastructure defined in your `.tf` files. Terraform will print the plan and prompt you to type `yes` before making any changes. |
| `terraform destroy` | Tears down all infrastructure managed by Terraform in this directory. You will be prompted to confirm. Run this when you are done with the lab to avoid ongoing AWS charges. Before redeploying, verify the destroy completed cleanly: check your [AWS EC2 Dashboard](https://console.aws.amazon.com/ec2/home) and confirm all redStack instances show as terminated and no Elastic IPs remain allocated. |

For full command reference, see the [Terraform CLI documentation](https://developer.hashicorp.com/terraform/cli/commands).

</details>

<details>
<summary><strong>AWS EC2 Dashboard Primer:</strong> not familiar with the AWS Console? Click to expand.</summary>

The [AWS EC2 Dashboard](https://console.aws.amazon.com/ec2/home) is your primary visibility tool for what Terraform has built (or destroyed) in AWS. You will use it to verify deployments and confirm clean teardowns. Key sections:

| Section | Where to find it | What to check |
| --- | --- | --- |
| **Instances** | EC2 > Instances > Instances | All 6 redStack instances should show `running` after `terraform apply`. After `terraform destroy`, all should show `terminated`. |
| **Elastic IPs** | EC2 > Network & Security > Elastic IPs | Two EIPs are allocated at deploy time (Guacamole, Redirector). After `terraform destroy`, both should be released (not listed). Unreleased EIPs incur charges. |
| **Key Pairs** | EC2 > Network & Security > Key Pairs | Confirm `rs-rsa-key` exists before deploying. Terraform does not create this. It must be present or `terraform apply` will fail. |
| **VPCs** | VPC > Your VPCs | Two VPCs are created: TeamServer VPC (`172.31.0.0/16`) and Redirector VPC (`10.60.0.0/16`). After destroy, both should be gone. |

**Quick region check:** Make sure the AWS Console region (top-right dropdown) matches the region in your `terraform.tfvars` (`us-east-1` by default). Resources created in one region are invisible when viewing another.

</details>

---

> [!CAUTION]
> **AWS Cost Warning: Unattended Instances**
>
> Running EC2 instances accrue charges 24/7 whether you are actively using them or not. Forgetting about a deployed lab is one of the most common causes of unexpected AWS bills. A few things to keep in mind:
>
> - **Recommended pattern: `terraform destroy` after each study session.** For a 5 to 10 hrs/wk study cadence, destroying between sessions costs ~$5 to $9/mo (compute only) versus ~$26 to $30/mo for a stopped-but-not-destroyed lab. Redeploy in ~30 to 45 min next time.
> - **Stopping instances does not eliminate charges.** EBS volumes (~$14/mo) and Elastic IPs (~$7/mo) bill 24/7 even when instances are stopped. That is ~$21/mo of fixed cost for a paused deployment.
> - **`terraform destroy` is the only way to zero out charges.** It terminates instances, releases Elastic IPs, and removes EBS volumes.
> - **Set a billing alarm.** In the [AWS Billing Console](https://console.aws.amazon.com/billing/home), create a CloudWatch billing alarm to alert you if monthly charges exceed a threshold you set. This is the best safeguard against runaway costs from forgotten resources.
>
> See [Cost Management](#cost-management) below for the full breakdown across destroy/redeploy, stopped, and 24/7 scenarios.

### Step 1.2: Initialize Terraform

```bash
terraform init
```

**Expected Output:**

```bash
Initializing the backend...
Initializing provider plugins...
- Finding hashicorp/aws versions matching "~> 5.0"...
- Installing hashicorp/aws v5.x.x...
Terraform has been successfully initialized!
```

**Checkpoint:** ✅ No errors, providers downloaded

### Step 1.3: Review Deployment Plan

```bash
terraform plan
```

<details>
<summary>Expected output</summary>

- **~50+ resources** to be created
- 6 EC2 instances (Mythic, Sliver, Havoc, Guacamole, Windows, Redirector)
- 2 VPCs (team server VPC + redirector VPC)
- 2 Elastic IPs (Guacamole, Redirector) - **static, persistent IPs**
- Mythic, Sliver, and Havoc have **no public IPs** (internal only)
- Security groups, VPC peering, route tables
- No errors or warnings

</details>

**Checkpoint:** ✅ Plan shows expected resources

### Step 1.4: Deploy Infrastructure

```bash
terraform apply
```

Type `yes` when prompted.

**Expected Output:**

```bash
Apply complete! Resources: 50+ added, 0 changed, 0 destroyed.
```

**Checkpoint:** ✅ Terraform apply completed successfully

### Step 1.5: Review Deployment Information

There are two outputs: `deployment_info` (all IPs, credentials, SSH commands) and `network_architecture` (diagram with actual IPs).

```bash
terraform output deployment_info
terraform output network_architecture
```

> [!TIP]
> Save `deployment_info` to a file for quick offline reference. You will need the IPs, credentials, and C2 header throughout this guide.

**Checkpoint:** ✅ Deployment info reviewed, IPs and credentials noted

### Step 1.6: Point Domain to Redirector

> [!NOTE]
> **Closed environment (no DNS):** Skip this step entirely. No domain or DNS record is needed. Proceed to Part 2.

After deployment, you need to point your domain's DNS to the redirector's Elastic IP so that Certbot can obtain a valid SSL certificate.

**Get the Redirector IP:**

```bash
terraform output deployment_info
```

Look for the **APACHE REDIRECTOR** section. The `Public IP` field is what you need.

**Create DNS A Records:**

Log into your domain registrar or DNS provider (e.g., Namecheap, Cloudflare, Route53) and create A records pointing to the redirector's Elastic IP. Use whichever host matches how you set `redirector_domain` in `terraform.tfvars`:

| Type | Host | Value | TTL |
| ---- | ---- | ----- | --- |
| A Record | `@` | `<Redirector Elastic IP>` | Automatic |
| A Record | `www` | `<Redirector Elastic IP>` | Automatic |
| A Record | `sub` | `<Redirector Elastic IP>` | Automatic |

Only `@` (the apex domain) is required. Add `www` only if you want callbacks over `www.yourdomain.tld`.

The `sub` row is a placeholder for any custom subdomain you want to use, for example `test.yourdomain.tld`, `cdn.yourdomain.tld`, or `chat.yourdomain.tld`. Custom subdomains blend beacon and implant traffic into patterns that look like legitimate service traffic. That makes callbacks harder to flag in firewall logs and network monitoring, which simulates real-world C2 tradecraft so you can practice detection and evasion in a lab setting.

**Verify DNS Propagation** (substitute your actual `redirector_domain` value from `terraform.tfvars`):

<details>
<summary>Windows (PowerShell)</summary>

```powershell
Resolve-DnsName yourdomain.tld
```

</details>

<details>
<summary>Linux/Mac (bash)</summary>

```bash
dig +short yourdomain.tld
```

</details>

**Expected:** The IP returned should match your redirector's Elastic IP.

> [!NOTE]
> DNS propagation can vary. Once DNS resolves correctly, proceed to Part 3 to run Certbot and configure the redirector.

**Checkpoint:** ✅ Domain pointed to redirector IP, DNS verified

> [!NOTE]
> **Wait 5-10 minutes before proceeding to Part 2.** User data scripts are installing software on all servers:
>
> - All hosts: Setting descriptive hostnames and populating `/etc/hosts` for cross-host name resolution
> - Mythic (`mythic`): Installing Docker, cloning Mythic, starting ~10 containers (Debian 12)
> - Sliver (`sliver`): Installing Sliver C2 server binary, configuring firewall (Debian 12)
> - Havoc (`havoc`): Installing Go, cloning and building Havoc teamserver from source (Debian 12)
> - Guacamole (`guac`): Setting up PostgreSQL, Nginx, Docker containers (Debian 12)
> - Windows (`WIN-OPERATOR`): Disabling Defender/Firewall, enabling RDP, installing tools (Chromium, VS Code, MobaXterm, 7-Zip)
> - Redirector (`redirector`): Installing Apache with mod_rewrite/proxy, configuring header+URI validation, downloading redirect.rules, setting up SSL and decoy page (Debian 12, fully automated)

---

## Part 2: Verification

Verify all components are accessible before moving to C2 setup. All credentials and IPs come from:

```bash
terraform output deployment_info
```

> **Pre-configured hostnames** are written to `/etc/hosts` on every Linux machine and `C:\Windows\System32\drivers\etc\hosts` on Windows during deployment. Use hostnames (`mythic`, `sliver`, `havoc`, `guac`, `redirector`, `win-operator`) instead of IPs from anywhere inside the lab.

### Step 2.1: Access Guacamole Portal

Open in your browser:

```http
https://<GUAC_PUBLIC_IP>/guacamole
```

- Username: `guacadmin`
- Password: from `terraform output deployment_info`

After logging in you should see **7 pre-configured connections**:

```yaml
Step 1: Windows Operator Workstation  (RDP) - auto-connects with Administrator credentials
Step 2: Mythic C2 Server              (SSH)
Step 3: Guacamole Server              (SSH)
Step 4: Apache Redirector             (SSH)
Step 5: Sliver C2 Server              (SSH)
Step 6: Havoc C2 Server               (SSH)
Step 7: Havoc C2 Desktop              (VNC) - XFCE4 desktop with Havoc GUI client
```

All SSH connections use password auth (no keys needed) pre-configured with the auto-generated lab password.

**Checkpoint:** ✅ Guacamole accessible, 7 connections visible

### Step 2.2: Access Windows Workstation

```yaml
Step 1: Click "Windows Operator Workstation"
Step 2: RDP connects automatically - wait 10-30 seconds for the desktop to load
Step 3: Verify installed - Chromium, VS Code, MobaXterm, 7-Zip
Step 4: Open MobaXterm - the redStack Lab folder has pre-configured SSH sessions for all lab machines
```

**If the connection fails:** Wait 5 more minutes. Windows is the slowest component to initialize.

**Checkpoint:** ✅ Windows desktop accessible, tools present, MobaXterm sessions visible

### Step 2.3: Verify Internal Connectivity

From the Windows workstation open **PowerShell** and ping the lab machines to confirm hostname resolution and network connectivity:

```powershell
ping mythic
ping sliver
ping havoc
ping redirector
ping guac
```

**Expected:** All hostnames resolve and respond.

**Checkpoint:** ✅ All lab machines reachable by hostname from Windows

---

## Operating the Lab (Wiki)

The README ends here. From this point forward, lab operation lives in the [redStack wiki](https://github.com/BaddKharma/redStack/wiki).

The wiki covers the deep-dive material that used to occupy ~1,200 lines in the README. Pick the page that matches what you are about to do.

### Per-C2 walkthroughs

Each C2 framework has its own page covering server access, listener setup, payload generation, and getting an initial beacon back to the Windows operator workstation through the redirector.

- **[Mythic C2 Setup](https://github.com/BaddKharma/redStack/wiki/Mythic-C2)** — Pre-installed profiles, agent generation, Apollo deployment, callback verification
- **[Sliver C2 Setup](https://github.com/BaddKharma/redStack/wiki/Sliver-C2)** — Server access, C2 profile import, HTTP listener, implant generation, callback verification
- **[Havoc C2 Setup](https://github.com/BaddKharma/redStack/wiki/Havoc-C2)** — Build script, teamserver, desktop client, listener, demon generation, callback verification

### Operator workstations

- **[Kali Operator](https://github.com/BaddKharma/redStack/wiki/Kali-Operator)** — Headless vs GUI mode, the curated 21-package AD/enum tool installer, SSH port forwarding for Meterpreter callbacks, AD attack workflows

### Infrastructure deep dives

- **[Apache Redirector](https://github.com/BaddKharma/redStack/wiki/Apache-Redirector)** — SSL certificate management, three-layer security model (redirect.rules + header + URI), log review
- **[External Target Environments (HTB / VulnLab / Proving Grounds)](https://github.com/BaddKharma/redStack/wiki/External-Targets)** — OpenVPN + WireGuard routing for closed-environment Pro Lab work

### Reference

- **[Troubleshooting](https://github.com/BaddKharma/redStack/wiki/Troubleshooting)** — Connectivity checks, component health, fixes for common failures (Mythic SSL, Sliver missing, Havoc build, agent callbacks not arriving, Terraform errors)
## Post-Deployment Actions

### Cleanup (When Done)

```bash
terraform destroy
# Type 'yes' to confirm

# Verify removal
aws ec2 describe-instances --filters "Name=tag:Project,Values=redstack"
```

### Cost Management

All numbers below are for `us-east-1` on-demand pricing with the default redStack instance mix (2x Linux `t3.medium`, 3x Linux `t3.small`, 1x Windows `t3.medium`).

#### Recommended: Destroy Between Sessions

For a 5 to 10 hrs/wk study cadence, the cheapest pattern is to run `terraform destroy` at the end of each session and `terraform apply` again next time. You pay only for the hours the lab is actually running. No idle EBS, no idle Elastic IPs, no licensed Windows volume sitting around.

**Hourly compute mix:**

| Instance | Rate | Count | Subtotal |
| --- | --- | --- | --- |
| Linux `t3.medium` (Mythic, Havoc) | $0.0416/hr | 2 | $0.0832 |
| Linux `t3.small` (Guacamole, Sliver, Redirector) | $0.0208/hr | 3 | $0.0624 |
| Windows `t3.medium` (WIN-OPERATOR, license included) | $0.0608/hr | 1 | $0.0608 |
| **Total** | | | **~$0.21/hr** |

**Monthly cost at this cadence:**

| Hours/week | Hours/month | Compute | EBS + EIPs (no idle) | **Total** |
| --- | --- | --- | --- | --- |
| 5 hrs/wk | ~22 hrs | ~$5 | $0 | **~$5/mo** |
| 10 hrs/wk | ~43 hrs | ~$9 | $0 | **~$9/mo** |

**Tradeoffs to know about:**

- **First-time setup per redeploy is ~30 to 45 min** before the lab is ready to use: `terraform apply` (~10 min) + cloud-init (~5 to 10 min) + Windows initialization (up to ~15 min) + Havoc rebuild from source (~15 to 25 min, only on first deploy or if you redeploy from scratch).
- **DNS A records change** each redeploy in open mode (the redirector EIP is new each time). Update your registrar after every `apply`. With a short DNS TTL this propagates in minutes.
- **Let's Encrypt** has a 5 duplicate-cert-per-week limit per registered domain. Two to three sessions per week is fine. If you redeploy more often, switch to a staging cert during testing or use the closed-environment self-signed flow.
- **In-progress C2 state is destroyed** with the lab. Mythic callbacks, Sliver implants, Havoc demons, custom listener configs are all gone. For training that is usually the point. For longer engagements, use the stopped pattern below.

#### Alternative: Stop Between Sessions

Faster to resume (~5 to 10 min instance boot vs ~30 to 45 min full redeploy) and preserves all C2 state, at the cost of ~$21/mo of always-on storage and Elastic IPs.

| Hours/week | Compute | EBS + EIPs (always on) | **Total** |
| --- | --- | --- | --- |
| 5 hrs/wk | ~$5 | $21 | **~$26/mo** |
| 10 hrs/wk | ~$9 | $21 | **~$30/mo** |

The fixed $21/mo is roughly $14 EBS gp3 (175 GB total across 6 volumes) + $7 for the 2 Elastic IPs (Guacamole and redirector, billed 24/7 since the 2024 pricing change).

#### Reference: Always On

| Item | Calculation | Cost |
| --- | --- | --- |
| Compute, all instances 24/7 | 730 hrs x $0.2064/hr | ~$151/mo |
| EBS gp3 storage | 175 GB x $0.08/GB-mo | ~$14/mo |
| Elastic IPs | 2 x $0.005/hr x 730 hrs | ~$7/mo |
| **Total** | | **~$172/mo** |

Data transfer is excluded from all three scenarios because the first 100 GB/mo of outbound traffic is free and light study use does not exceed it.

> [!IMPORTANT]
> EBS volumes and Elastic IPs bill 24/7 even when instances are stopped. Stopping pauses compute charges only. The only way to eliminate all charges is `terraform destroy`.

#### How to Run Each Pattern

**Destroy after each session (recommended):**

```bash
terraform destroy
# Type 'yes' to confirm
```

Next session, redeploy with `terraform apply`. Update your DNS A records to the new redirector EIP and re-run Certbot in open mode.

**Stop instances (alternative, preserves state):**

```yaml
Step 1: AWS Console > EC2 > Instances
Step 2: Select all redStack instances
Step 3: Instance State > Stop
```

Or via AWS CLI (get instance IDs from `aws ec2 describe-instances`):

```bash
aws ec2 stop-instances --instance-ids i-xxxxx i-yyyyy i-zzzzz
```

---

## Success Criteria

At the end of this deployment, you should have:

- ✅ All 6 EC2 instances running and accessible
- ✅ 2 VPCs with peering configured
- ✅ Apache redirector with header validation, URI routing, and redirect.rules
- ✅ Mythic C2 server isolated (no public IP, internal only)
- ✅ Sliver C2 server isolated (no public IP, internal only)
- ✅ Havoc C2 server isolated (no public IP, internal only)
- ✅ Mythic HTTP listener configured through redirector (/cdn/media/stream/)
- ✅ Sliver HTTP listener configured through redirector (/cloud/storage/objects/)
- ✅ Havoc HTTP listener configured through redirector (/edge/cache/assets/)
- ✅ Header validation (X-Request-ID) filtering unauthorized requests
- ✅ redirect.rules blocking AV vendors and TOR exits (403); cloud IPs commented out for AWS compatibility
- ✅ Decoy page served for requests without valid C2 header
- ✅ At least 1 active agent callback per C2 framework
- ✅ Can execute commands through all C2 paths
- ✅ Guacamole connections auto-created (1 RDP for Windows, 6 SSH for the Linux hosts, 1 VNC for Havoc desktop, plus 1 XRDP for Kali when `kali_deployment_mode = "gui"`)
- ✅ Guacamole providing web-based access to all infrastructure components
- ✅ SSH password authentication working from C2 VPC, keys required from public IPs
- ✅ Windows workstation with Chromium, VS Code, MobaXterm, and 7-Zip installed
- ✅ Windows Administrator accessible via Guacamole (AWS-generated password auto-configured)
- ✅ Kali operator workstation accessible via Guacamole; MOTD banner with `install-kali-tools` and `kali-go-gui` hints visible on first SSH login

**Your Boot-to-Breach lab environment is operational.** When you are ready to start operating it, head to the [redStack wiki](https://github.com/BaddKharma/redStack/wiki) for per-C2 walkthroughs, the Kali tooling, the Apache redirector deep dive, and external-target VPN routing.

