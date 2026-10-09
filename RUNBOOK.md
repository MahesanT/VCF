# VCF 9.0 Holodeck Deployment Runbook

> This is the command-focused companion to the illustrated GitHub Pages walkthrough. It records the key commands and configuration used in the lab. Secrets are intentionally omitted.

## 1. Lab architecture

| Component | Address / Detail |
|---|---|
| Physical ESXi 1 | `192.168.8.220` — Dell PowerEdge T440, 40 physical cores / 80 logical, ~318.6 GB RAM |
| Physical ESXi 2 | `192.168.8.45` — Dell Precision T7910, 36 physical cores / 72 logical, ~159.9 GB RAM |
| vCenter | `192.168.8.221` |
| HoloRouter | `192.168.8.222` |
| Offline Depot | `192.168.8.223` |
| NFS VM | `192.168.8.214` / `10.10.10.10` |
| Shared datastore | `Holo-Shared` |
| VCF Installer | `10.1.10.250` |
| SDDC Manager | `10.1.1.5` / `sddcmanager-a.site-a.vcf.lab` |
| VCF version | `9.0.0.0` |
| Holodeck | 9.0 |
| VCF Download Tool | `9.1.1.0.25713929` |

## 2. Physical ESXi preparation

Both physical hosts were upgraded to ESXi 8.0 U3 before building the nested lab. Configuration backups were taken first.

### Holodeck trunk vSwitch

The Holodeck trunk was placed on a direct 10 Gb NIC (`vmnic2`) on both physical hosts.

```bash
esxcli network vswitch standard add -v vSwitch-Holo
esxcli network vswitch standard set -v vSwitch-Holo -m 9000
esxcli network vswitch standard uplink add -v vSwitch-Holo -u vmnic2

esxcli network vswitch standard portgroup add -v vSwitch-Holo -p Holo-Trunk
esxcli network vswitch standard portgroup set -p Holo-Trunk -v 4095

esxcli network vswitch standard portgroup policy security set -p Holo-Trunk -a true
esxcli network vswitch standard portgroup policy security set -p Holo-Trunk -m true
esxcli network vswitch standard portgroup policy security set -p Holo-Trunk -f true
```

The security settings are required for nested ESXi / VLAN trunking:
- Promiscuous mode: Accept
- MAC address changes: Accept
- Forged transmits: Accept

## 3. Shared NFS datastore over direct 10 Gb

A second direct 10 Gb link (`vmnic3`) was dedicated to NFS.

### ESXi 1

```bash
esxcli network vswitch standard add -v vSwitch-NFS
esxcli network vswitch standard set -v vSwitch-NFS -m 9000
esxcli network vswitch standard uplink add -v vSwitch-NFS -u vmnic3

esxcli network vswitch standard portgroup add -v vSwitch-NFS -p NFS-Network
esxcli network ip interface add -i vmk1 -p NFS-Network
esxcli network ip interface ipv4 set -i vmk1 -I 10.10.10.1 -N 255.255.255.0 -t static
esxcli network ip interface set -i vmk1 -m 9000
```

### ESXi 2

```bash
esxcli network vswitch standard add -v vSwitch-NFS
esxcli network vswitch standard set -v vSwitch-NFS -m 9000
esxcli network vswitch standard uplink add -v vSwitch-NFS -u vmnic3

esxcli network vswitch standard portgroup add -v vSwitch-NFS -p NFS-Network
esxcli network ip interface add -i vmk1 -p NFS-Network
esxcli network ip interface ipv4 set -i vmk1 -I 10.10.10.2 -N 255.255.255.0 -t static
esxcli network ip interface set -i vmk1 -m 9000
```

Jumbo-frame validation:

```bash
vmkping -I vmk1 -s 8972 -d 10.10.10.2
# reverse direction from the other host
vmkping -I vmk1 -s 8972 -d 10.10.10.1
```

### Ubuntu NFS server

A Ubuntu Server VM was created on the T440 with:
- 2 vCPU
- 4 GB RAM
- 32 GB OS disk
- 1.4 TB data disk
- `ens34 = 10.10.10.10/24`, MTU 9000

Install NFS:

```bash
sudo apt update
sudo apt install -y nfs-kernel-server
```

Data filesystem:

```bash
sudo parted /dev/sdb --script mklabel gpt
sudo parted /dev/sdb --script mkpart primary ext4 0% 100%
sudo mkfs.ext4 -L nfsdata /dev/sdb1
sudo mkdir -p /srv/nfs/holodeck
sudo mount /dev/sdb1 /srv/nfs/holodeck
```

Persistent mount:

```text
UUID=<filesystem-uuid> /srv/nfs/holodeck ext4 defaults,nofail 0 2
```

`/etc/exports`:

```text
/srv/nfs/holodeck 10.10.10.1(rw,sync,no_subtree_check,no_root_squash) 10.10.10.2(rw,sync,no_subtree_check,no_root_squash)
```

Apply:

```bash
sudo exportfs -ra
sudo exportfs -v
```

Mount from both ESXi hosts:

```bash
esxcli storage nfs add -H 10.10.10.10 -s /srv/nfs/holodeck -v Holo-Shared
```

## 4. vCenter and cluster

vCenter 8.0 U3 was deployed at `192.168.8.221`.

Objects:
- Datacenter: `Homelab-DC`
- Cluster: `HomeLab-Cluster`
- DRS: Off
- HA: Off
- vSAN: Off at the physical cluster layer
- Shared datastore: `Holo-Shared`

## 5. HoloRouter

HoloRouter was deployed from `holorouter-9.0.0.ova`.

Key settings:
- Management IP: `192.168.8.222/24`
- Gateway: `192.168.8.1`
- Site A trunk: `Holo-Trunk`
- Site B trunk: `Holo-Trunk`
- DNS domain: `site-a.vcf.lab`
- SSH enabled

Verify PowerShell / Holodeck module:

```powershell
pwsh
Get-Command New-HoloDeckInstance -Syntax
Get-Command New-HoloDeckConfig -Syntax
```

A Holodeck configuration was created and stored under:

```text
/holodeck-runtime/config/5rz8.json
```

Target:
- vCenter API type: `VirtualCenter`
- Datacenter: `Homelab-DC`
- Cluster: `HomeLab-Cluster`
- Datastore: `Holo-Shared`
- Network: `Holo-Trunk`
- Deprecated CPU override: enabled for the nested lab

Precheck:

```powershell
Start-HoloDeckPrecheck -Version "9.0"
```

## 6. Stage VCF binaries required by Holodeck

The following were placed in:

```text
/holodeck-runtime/bin/9.0/
```

- ESXi 9.0 installer ISO
- VCF SDDC Manager / Installer appliance OVA

Provision nested infrastructure only:

```powershell
New-HoloDeckInstance `
  -Version "9.0" `
  -ManagementOnly `
  -ProvisionOnly
```

The first attempt exposed a URI parsing issue because a vCenter password contained URL-sensitive characters. The password was changed to a safer lab password and the Holodeck config updated. The rerun succeeded.

Result:
- `pink-esx-01a`
- `pink-esx-02a`
- `pink-esx-03a`
- `pink-esx-04a`
- `pink-vcfinstaller-a`

Generated management-domain spec:

```text
/holodeck-runtime/specs/management_domain_config_spec_for_vcf_installer.json
```

## 7. Access the VCF Installer from the Windows jump host

VCF Installer address:

```text
10.1.10.250
```

Persistent Windows route through HoloRouter:

```powershell
route -p add 10.1.10.0 mask 255.255.255.0 192.168.8.222
route -p add 10.1.1.0 mask 255.255.255.0 192.168.8.222
```

## 8. Offline Depot appliance

Offline Depot VM:
- `192.168.8.223`
- nginx web root: `/var/www/build`

The bundled download tool was older:

```bash
sudo /root/vdt/bin/vcf-download-tool --version
```

A newer VCF Download Tool was copied and extracted side-by-side:

```bash
mkdir -p ~/vdt-9.1.1
tar -xzf ~/vcf-download-tool-9.1.1.0.25713929.tar.gz -C ~/vdt-9.1.1
~/vdt-9.1.1/bin/vcf-download-tool --version
```

Version used:

```text
9.1.1.0.25713929
```

### Software Depot ID and activation code

Generate a Software Depot ID:

```bash
~/vdt-9.1.1/bin/vcf-download-tool configuration generate --software-depot-id
```

The Software Depot ID was registered at the Broadcom VCF portal and an activation code was generated.

Store the activation code securely:

```bash
read -s ACTIVATION_CODE
printf '%s' "$ACTIVATION_CODE" > /home/admin/vcf-activation-code.txt
unset ACTIVATION_CODE
chmod 600 /home/admin/vcf-activation-code.txt
```

### List required VCF 9.0 install binaries

```bash
~/vdt-9.1.1/bin/vcf-download-tool binaries list \
  --depot-download-activation-code-file=/home/admin/vcf-activation-code.txt \
  --vcf-version=9.0.0 \
  --sku=VCF \
  --automated-install \
  --type=INSTALL
```

Seven Day-0 components were returned:
- SDDC Manager
- VCF Automation
- VCF Operations
- VCF Operations Fleet Management
- NSX
- VCF Operations Cloud Proxy / Collector
- vCenter

### Download the full set

```bash
mkdir -p /home/admin/vcf90-depot

~/vdt-9.1.1/bin/vcf-download-tool binaries download \
  --depot-download-activation-code-file=/home/admin/vcf-activation-code.txt \
  --depot-store=/home/admin/vcf90-depot \
  --vcf-version=9.0.0.0 \
  --sku=VCF \
  --automated-install \
  --type=INSTALL
```

All 7 binaries completed successfully (~53 GB).

Copy into nginx:

```bash
sudo mkdir -p /var/www/build/PROD
sudo cp -a /home/admin/vcf90-depot/PROD/. /var/www/build/PROD/
sudo chown -R nginx:nginx /var/www/build
sudo chmod -R 750 /var/www/build
sudo chmod g+s /var/www/build
```

Verify catalog:

```bash
curl -I http://127.0.0.1/PROD/metadata/productVersionCatalog/v1/productVersionCatalog.json
```

Expected:

```text
HTTP/1.1 200 OK
```

## 9. Allow HTTP Offline Depot in VCF Installer

The VCF 9.0 Installer UI initially required HTTPS.

As `root` on `vcfinstaller-a`:

```bash
cp /opt/vmware/vcf/lcm/lcm-app/conf/application-prod.properties \
   /opt/vmware/vcf/lcm/lcm-app/conf/application-prod.properties.bak

echo 'lcm.depot.adapter.httpsEnabled=false' >> \
/opt/vmware/vcf/lcm/lcm-app/conf/application-prod.properties

systemctl restart lcm
systemctl status lcm
```

The Offline Depot was then configured as:
- Host: `192.168.8.223`
- Port: `80`

The three previously failed online downloads (Automation, Fleet Management, NSX) were retried from the Offline Depot and succeeded.

## 10. Import the Holodeck JSON spec

Copy the generated spec to the Windows machine:

```powershell
scp root@192.168.8.222:/holodeck-runtime/specs/management_domain_config_spec_for_vcf_installer.json `
  "$env:USERPROFILE\Downloads\management_domain_config_spec_for_vcf_installer.json"
```

In VCF Installer:
1. **Deploy Using JSON Spec**
2. Upload the file
3. Review generated design
4. Run validation

## 11. VCF 9.0 management-domain design

| Network | VLAN | CIDR | MTU |
|---|---:|---|---:|
| ESXi Management | 10 | `10.1.1.0/24` | 1500 |
| vSAN | 11 | `10.1.2.0/25` | 8000 |
| vMotion | 13 | `10.1.3.0/25` | 8000 |
| NSX TEP | 14 | `10.1.3.128/25` | 8000 |

Core components:
- vCenter: `vc-mgmt-a.site-a.vcf.lab`
- NSX: `nsx-mgmt-a.site-a.vcf.lab`
- SDDC Manager: `sddcmanager-a.site-a.vcf.lab`
- VCF Operations: `ops-a.site-a.vcf.lab`
- Fleet Management: `opslcm-a.site-a.vcf.lab`
- Cluster: `cluster-mgmt-01a`
- vSAN datastore: `vsan-mgmt-01a`
- vSAN architecture: OSA
- Four nested ESXi hosts

## 12. Fix DNS and NTP validation warnings

The generated spec expected `10.1.1.1`, while VCF Installer initially used `10.1.10.129`.

Persistent DNS configuration was in:

```text
/etc/systemd/network/10-eth0.network
```

Back up and update:

```bash
cp /etc/systemd/network/10-eth0.network \
   /etc/systemd/network/10-eth0.network.bak

cp /etc/ntp.conf /etc/ntp.conf.bak

sed -i 's/^DNS=10\.1\.10\.129$/DNS=10.1.1.1/' \
/etc/systemd/network/10-eth0.network

sed -i 's/^server 10\.1\.10\.129$/server 10.1.1.1/' \
/etc/ntp.conf

networkctl reload
networkctl reconfigure eth0
systemctl restart systemd-resolved
systemctl restart ntpd
```

Verify:

```bash
resolvectl status eth0
grep '^server' /etc/ntp.conf
ntpq -pn
```

The NTP peer eventually showed:

```text
*10.1.1.1
```

After rerunning VCF validation, all checks passed.

## 13. Deploy the management domain

The VCF Installer completed:
1. Deploy vCenter
2. Deploy SDDC Manager
3. Configure vSphere cluster
4. Deploy and configure NSX
5. Deploy and configure Fleet Management appliance
6. Deploy and configure VCF Operations appliance

Final result: **100% complete / deployment successful**.

## 14. Post-deployment access

SDDC Manager:

```text
sddcmanager-a.site-a.vcf.lab -> 10.1.1.5
```

Windows hosts-file entry:

```text
10.1.1.5 sddcmanager-a.site-a.vcf.lab
```

Flush DNS:

```powershell
ipconfig /flushdns
```

## 15. Post-deployment health

SDDC Manager:
- Management domain `mgmt-a`: Active
- Four ESXi hosts
- vCenter / NSX / vSAN operational
- Evaluation licensing warnings only

VCF Operations:
- 1 VCF instance
- 4 ESXi hosts
- 1 vCenter
- 1 vSAN cluster
- 1 NSX instance
- Status: **Good**
- Critical issues: **0**

## 16. Useful deployment logs

Primary VCF deployment log:

```bash
tail -F /var/log/vmware/vcf/domainmanager/domainmanager.log
```

Filtered:

```bash
tail -F /var/log/vmware/vcf/domainmanager/domainmanager.log \
  | grep --line-buffered -Ei 'ERROR|WARN|FAILED|SUCCESS|deploy|workflow|task'
```

Lifecycle Manager:

```bash
tail -F /var/log/vmware/vcf/lcm/lcm.log
```

Find vCenter CLI installer logs:

```bash
find /var/log/vmware/vcf/domainmanager \
  -name vcsa-cli-installer.log -type f
```

## 17. Security / cleanup notes

Do not commit:
- Broadcom Download Tokens
- Activation Codes
- VCF generated passwords
- vCenter / ESXi / appliance passwords

The lab uses RFC1918 addresses and `.lab` hostnames. Rotate any credentials that were ever exposed in terminal screenshots.
