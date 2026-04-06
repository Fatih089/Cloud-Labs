# Lab N1 SNAT Exhaustion NAT Gateway vs Load Balancer

Dauer: 60–90 min
Ziel: Verstehen, warum SNAT‑Ports knapp werden, wie NAT Gateway das Problem löst und warum ein Standard Load Balancer allein nicht reicht.

1 Umgebung vorbereiten
1. Resource Group erstellen  
Portal → Resource groups → Create
Name: rg-n1-snat
Region: West Europe
Create
2. VNet + Subnet erstellen  
Portal → Virtual networks → Create
Name: vnet-n1
Address space: 10.10.0.0/16
Subnet: vm-subnet → 10.10.1.0/24
Create
2 VM Scale Set ohne NAT Gateway deployen
Portal → Virtual machine scale sets → Create
Name: vmss-n1
Region: West Europe
Image: Ubuntu LTS
Instances: 2
Networking:
VNet: vnet-n1
Subnet: vm-subnet
Public IP: None
Load Balancer: Standard LB automatisch erstellen lassen
Create
Erwartung: VMSS nutzt den Standard Load Balancer für Outbound SNAT.
3 SNAT‑Exhaustion provozieren
Auf einer VMSS‑Instanz per SSH (über Bastion oder Jumpbox):
bash
for i in {1..5000}; do curl -m 1 https://microsoft.com >/dev/null 2>&1 & done
Was du beobachten wirst:
Manche Verbindungen schlagen fehl
Im LB fehlen Outbound Rules → SNAT Ports werden knapp
4 NAT Gateway hinzufügen
1. Public IP erstellen  
Portal → Public IP addresses → Create
Name: pip-nat
SKU: Standard
Assignment: Static
Create
2. NAT Gateway erstellen  
Portal → NAT gateways → Create
Name: nat-n1
Public IP: pip-nat
Create
3. Subnet mit NAT Gateway verknüpfen  
Portal → vnet-n1 → Subnets → vm-subnet → NAT gateway → nat-n1 → Save
5 Test wiederholen
Auf VMSS‑Instanz erneut:
bash
for i in {1..5000}; do curl -m 1 https://microsoft.com >/dev/null 2>&1 & done
Erwartung:
Keine SNAT‑Fehler mehr
Outbound Traffic stabil
Source IP = pip-nat
6 Validierung
CLI (optional):
bash
az network public-ip list -g rg-n1-snat -o table
az network nat gateway show -g rg-n1-snat -n nat-n1 -o table
Erwartet:
NAT Gateway ist Subnet‑assoziiert
Public IP sichtbar
VMSS Outbound stabil
7 Häufige Fehler + Fix
Fehler: Weiterhin SNAT‑Exhaustion
Fix: NAT Gateway wurde nicht korrekt mit Subnet verknüpft → Subnet‑Einstellungen prüfen.
8 Prüfungsfrage
Warum verhindert ein Standard Load Balancer allein keine SNAT‑Exhaustion?  
→ Weil er nur begrenzte SNAT‑Ports pro Frontend‑IP bereitstellt und keine automatische Skalierung wie ein NAT Gateway besitzt.
