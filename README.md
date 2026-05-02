<!-- markdownlint-disable MD060 MD040 -->

# redStack

![redStack Banner](assets/redStack-banner.png)

> A self-contained Boot-to-Breach lab on AWS. Deploy a full red-team training environment in ~30 minutes — three C2 frameworks (Mythic, Sliver, Havoc), an Apache redirector, a Kali operator, a Windows operator, and a Guacamole portal. Two peered VPCs, header + URI gating, scanner blocking, optional OpenVPN routing for HTB / VulnLab / Proving Grounds.

**📖 [Full documentation lives in the redStack Wiki →](https://github.com/BaddKharma/redStack/wiki)**

The wiki is the de facto operator handbook. This README is a thin landing page so you can find your way in. Everything you need to deploy, verify, run the C2 walkthroughs, troubleshoot, or extend the lab lives there.

---

> [!IMPORTANT]
> redStack is not a tutorial on how to use C2 frameworks. It is an environment that removes the infrastructure hurdle so you can focus on learning. **This lab is strictly for authorized training and lab environments only** (HTB, VL, PG, self-hosted cyber ranges, personal lab VMs, etc.). Not intended for use in real-world engagements or against targets you do not own and have explicit written permission to test.

> [!CAUTION]
> **AWS TOS: Use at your own risk.** Hosting C2 infrastructure on AWS may raise concerns under the [AWS Acceptable Use Policy](https://aws.amazon.com/aup/). Before deploying, review the AUP and submit the [AWS Penetration Testing / Simulated Events request form](https://aws.amazon.com/security/penetration-testing/). As long as you are using redStack exclusively for personal lab work and authorized training platforms, you are generally in the clear. To be safe, run redStack from a dedicated, single-purpose throwaway AWS account.

---

## Deploy in 5 Steps

| # | What | Where |
|---|------|-------|
| 1 | First-time AWS setup (account, IAM, CLI, SSH key, Kali Marketplace subscription) | **[Prerequisites](https://github.com/BaddKharma/redStack/wiki/Prerequisites)** |
| 2 | Pick open or closed environment | **[Deployment Modes](https://github.com/BaddKharma/redStack/wiki/Deployment-Modes)** |
| 3 | Configure tfvars and `terraform apply` | **[Deploy](https://github.com/BaddKharma/redStack/wiki/Deploy)** |
| 4 | Confirm Guacamole + Windows + internal DNS | **[Verify](https://github.com/BaddKharma/redStack/wiki/Verify)** |
| 5 | Land first beacon: pick a C2 | **[Mythic-C2](https://github.com/BaddKharma/redStack/wiki/Mythic-C2)** · **[Sliver-C2](https://github.com/BaddKharma/redStack/wiki/Sliver-C2)** · **[Havoc-C2](https://github.com/BaddKharma/redStack/wiki/Havoc-C2)** |

**Total time:** ~45-90 minutes on first deploy. Subsequent deploys: ~30-45 minutes.

**Returning operator?** [Quick-Start](https://github.com/BaddKharma/redStack/wiki/Quick-Start) is the abbreviated path.

---

## What Gets Deployed

Seven EC2 instances across two peered VPCs. Two have public Elastic IPs (Guacamole portal + Apache redirector); everything else is reachable only through Guacamole.

| Hostname | Role | Public IP |
|----------|------|-----------|
| `guac` | Guacamole portal (web SSH/RDP/VNC) | Yes |
| `redirector` | Apache reverse proxy + C2 frontend | EIP exposes 80/443 only |
| `mythic` | Mythic C2 server | No |
| `sliver` | Sliver C2 server | No |
| `havoc` | Havoc C2 server + desktop (VNC) | No |
| `WIN-OPERATOR` | Windows Server 2022 operator workstation | No |
| `kali` | Kali Linux operator (AD enum + attack toolset) | No |

Full inventory + sizing details: **[Lab-Inventory](https://github.com/BaddKharma/redStack/wiki/Lab-Inventory)**.
Architecture diagram: **[Lab-Architecture](https://github.com/BaddKharma/redStack/wiki/Lab-Architecture)**.

---

## Cost

Roughly **$0.25/hour** of compute while running. With `terraform destroy` between sessions (recommended), expected monthly cost is **~$5-11/month** for typical 5-10 hr/wk study cadence. Full breakdown including stop-vs-destroy tradeoffs: **[Cost-Management](https://github.com/BaddKharma/redStack/wiki/Cost-Management)**.

> [!CAUTION]
> Forgetting a deployed lab is the #1 cause of unexpected AWS bills. Set a CloudWatch billing alarm before your first `terraform apply`.

---

## When Something Breaks

**[Troubleshooting](https://github.com/BaddKharma/redStack/wiki/Troubleshooting)** covers the failure modes that actually come up: Mythic SSL cert, Sliver missing, Havoc build failed, agent not calling back, Marketplace `OptInRequired`, VPC limits, `redirect.rules` download issues, Kali user rename, `ssh -R` binding behavior, and more.

---

## Wiki Page Map

**Getting started:** [Home](https://github.com/BaddKharma/redStack/wiki) · [Quick-Start](https://github.com/BaddKharma/redStack/wiki/Quick-Start) · [Prerequisites](https://github.com/BaddKharma/redStack/wiki/Prerequisites) · [Deployment-Modes](https://github.com/BaddKharma/redStack/wiki/Deployment-Modes) · [Deploy](https://github.com/BaddKharma/redStack/wiki/Deploy) · [Verify](https://github.com/BaddKharma/redStack/wiki/Verify)

**Reference:** [Lab-Architecture](https://github.com/BaddKharma/redStack/wiki/Lab-Architecture) · [Lab-Inventory](https://github.com/BaddKharma/redStack/wiki/Lab-Inventory) · [SSH-Access](https://github.com/BaddKharma/redStack/wiki/SSH-Access) · [Cost-Management](https://github.com/BaddKharma/redStack/wiki/Cost-Management)

**C2 walkthroughs:** [Mythic-C2](https://github.com/BaddKharma/redStack/wiki/Mythic-C2) · [Sliver-C2](https://github.com/BaddKharma/redStack/wiki/Sliver-C2) · [Havoc-C2](https://github.com/BaddKharma/redStack/wiki/Havoc-C2)

**Operator workstations:** [Kali-Operator](https://github.com/BaddKharma/redStack/wiki/Kali-Operator)

**Infrastructure:** [Apache-Redirector](https://github.com/BaddKharma/redStack/wiki/Apache-Redirector) · [External-Targets](https://github.com/BaddKharma/redStack/wiki/External-Targets)

**Help:** [Troubleshooting](https://github.com/BaddKharma/redStack/wiki/Troubleshooting)

---

## Repository Layout

```
redStack/
├── *.tf                  Terraform infrastructure (variables, security groups, instances, outputs)
├── setup_scripts/        Cloud-init user-data scripts for each host
├── assets/               README banner and other static images
├── terraform.tfvars.example  Sample configuration with comments
└── README.md             This file
```

The Terraform code is intentionally flat (one file per role: `mythic.tf` would be a refactor target, but right now Mythic lives in `main.tf` alongside the shared bits; Sliver, Havoc, Redirector, and Kali each have their own files). Setup scripts in `setup_scripts/` are templated by Terraform `templatefile()` and rendered into user-data at apply time.

---

## License

MIT. See [LICENSE](LICENSE).

---

## Contributing

Issues and PRs welcome. The wiki is a separate Git repo (`redStack.wiki.git`) — clone it locally to edit pages directly. Screenshot and GIF placeholders throughout the wiki are filled in over time; if you've taken a useful capture of a redStack workflow, contributions there are especially welcome.
