# VMware Cloud Foundation 9.0 on Holodeck — Home Lab Build

A practical, end-to-end build of **VMware Cloud Foundation (VCF) 9.0** using **VMware Holodeck** on a two-host nested ESXi home lab.

This repository documents the actual build, troubleshooting, offline depot workflow, commands, network design, and post-deployment validation used to bring the management domain online.

## Highlights

- Two physical ESXi hosts with ~478 GB combined RAM
- vCenter 8.0 U3 management layer
- Holodeck 9.0 single-site management-domain deployment
- Four nested ESXi hosts
- Shared NFS datastore over a direct 10 Gb link
- VCF Offline Depot using VCF Download Tool 9.1.1
- VCF 9.0 management domain with vCenter, NSX, SDDC Manager, VCF Operations, Fleet Management, vSAN OSA
- Clean validation and successful deployment
- VCF Operations health status: **Good**, 0 critical issues

## GitHub Pages

The full illustrated walkthrough is in `docs/index.html`.

Public build documentation:

**https://mahesant.github.io/VCF/**

## Detailed runbook

See **[RUNBOOK.md](RUNBOOK.md)** for the command-by-command build notes.

## Security note

Passwords, Broadcom download tokens, activation codes, and generated VCF credentials are intentionally excluded from this repository. All published addresses and hostnames are anonymised examples. Original screenshots containing infrastructure identifiers are withheld pending redaction.

---

Built as a hands-on VCF 9.0 learning lab.
