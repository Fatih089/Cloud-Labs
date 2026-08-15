# 🖥️ 04 – Compute

Erweiterung der bestehenden Azure-Umgebung um einen zweiten Compute-Endpunkt sowie praktische Szenarien mit **Managed Disks, VM Extensions, VM Scale Sets und Azure App Service**.

Der Schwerpunkt lag auf reproduzierbarer VM-Konfiguration, Skalierung, Compute-Troubleshooting und der Weiterverwendung der bestehenden Netzwerk- und Storage-Architektur.

---

##  Compute-Architektur

```text
VM-A
 │
 │ Private Traffic
 │
 ▼
VNet Peering
 │
 ▼
VM-B
├── Private IP: 10.0.0.4
├── nsg-vm-b
├── Data Disk
│   └── /data
└── Custom Script Extension
        │
        ▼
   testostorageb
        │
        ▼
Blob Private Endpoint
172.16.1.4
```

`VM-A` und `VM-B` befinden sich in unterschiedlichen Virtual Networks und kommunizieren ausschließlich über den privaten Peering-Pfad.

---

##  Umgesetzt

- zweite Linux-VM `VM-B`
- keine Public IP für `VM-B`
- eigener Network Security Group Scope
- echter VM-zu-VM-Traffic über VNet Peering
- kontrolliertes NSG-Troubleshooting
- zusätzliche Managed Data Disk
- Partitionierung und Mount im Linux-Gastbetriebssystem
- Custom Script Extension
- Private-DNS-Erweiterung für `vnet-test`
- Zugriff auf bestehenden Blob Private Endpoint
- VM Scale Set mit Uniform Orchestration
- manueller Skalierungstest
- Autoscale-Regeln
- Windows App Service
- ASP.NET Framework 4.8
- Deployment Slot
- kostenbewusstes Cleanup temporärer Compute-Ressourcen

---

##  VM-B

| Einstellung | Konfiguration |
|---|---|
| Betriebssystem | Ubuntu Server 24.04 LTS |
| VM Size | `Standard_B2ts_v2` |
| Resource Group | `RG-A` |
| Region | West Europe |
| VNet | `vnet-test` |
| Subnet | `default` |
| Private IP | `10.0.0.4` |
| Public IP | Keine |
| NIC | `vm-b633` |
| NSG | `nsg-vm-b` |

VM-B wurde bewusst ohne Public IP bereitgestellt.

Administrative und technische Tests erfolgen über den bereits vorhandenen privaten Netzwerkpfad.

---

##  End-to-End VNet Peering

Das zuvor nur auf Routing-Ebene validierte VNet Peering wurde erstmals mit zwei echten Compute-Endpunkten getestet.

```text
VM-A
172.16.0.x
    │
    │ VNet Peering
    ▼
VM-B
10.0.0.4
```

Eine SSH-Verbindung von `VM-A` zu `VM-B` über die private IP war erfolgreich.

Damit wurden gemeinsam validiert:

- VNet Peering
- Azure Routing
- private IP-Kommunikation
- NSG-Verhalten
- tatsächlicher End-to-End-Traffic

---

##  NSG Troubleshooting

Für einen kontrollierten Fehler wurde auf `nsg-vm-b` temporär eine Regel erstellt, die SSH-Verbindungen aus dem Netzwerk von VM-A blockierte.

**Ergebnis:** Neue SSH-Verbindungen liefen in einen Timeout.

Nach Entfernen der Regel funktionierte der private SSH-Zugriff erneut.

Dadurch wurde der Fehler gezielt auf die Network Security Group eingegrenzt und der funktionierende Zustand anschließend wiederhergestellt.

---

##  Managed Data Disk

VM-B erhielt eine separate Managed Data Disk:

`vm-b-data01`

Konfiguration:

- Größe: `32 GiB`
- SKU: `Standard SSD LRS`
- Linux Device: `/dev/sdb`
- Partition: `/dev/sdb1`
- Mount Point: `/data`

Die Disk wurde nicht nur in Azure attached, sondern anschließend innerhalb des Gastbetriebssystems partitioniert und als nutzbares Dateisystem eingebunden.

```text
Managed Disk
    ↓
/dev/sdb
    ↓
/dev/sdb1
    ↓
/data
```

Damit bleiben Betriebssystem und zusätzliche Nutzdaten logisch getrennt.

---

##  Custom Script Extension

Für reproduzierbare Post-Deployment-Konfiguration wurde die **Custom Script Extension for Linux** verwendet.

Das finale Script:

`extension-test.sh`

wurde über den bestehenden Storage Account bereitgestellt und erzeugte erfolgreich:

`/opt/az104/extension-test.txt`

### Troubleshooting

Während der Umsetzung traten mehrere reale Fehler auf:

1. VM-B löste den Storage-Endpunkt zunächst öffentlich auf.
2. Der Blob-Zugriff endete dadurch mit HTTP `403`.
3. Ein Script war versehentlich im RTF-Format gespeichert.
4. `commandToExecute` referenzierte zeitweise den falschen Scriptnamen.
5. Eine NGINX-Installation scheiterte zusätzlich am fehlenden Outbound-Internetzugang der VM.

Die finale Extension wurde deshalb mit einem internetunabhängigen Testscript erfolgreich validiert.

---

##  Private DNS für VM-B

Der bestehende Blob Private Endpoint sollte auch aus `vnet-test` erreichbar sein.

Dafür wurde die bestehende Private DNS Zone:

`privatelink.blob.core.windows.net`

zusätzlich über:

`link-vnet-test-blob`

mit `vnet-test` verbunden.

Vorher:

```text
testostorageb.blob.core.windows.net
        ↓
Public IP
```

Nach der DNS-Erweiterung:

```text
testostorageb.blob.core.windows.net
        ↓
Private Link
        ↓
172.16.1.4
```

Damit konnte die vorhandene Private-Endpoint-Architektur ohne zusätzlichen Storage Endpoint weiterverwendet werden.

---

##  Virtual Machine Scale Set

Temporär wurde `VMSS-WEB` mit **Uniform Orchestration** erstellt.

### Konfiguration

- Ubuntu Server 24.04 LTS
- `Standard_D2als_v7`
- Availability Zones 1, 2 und 3
- Ausgangskapazität: 2 Instanzen

### Manueller Skalierungstest

```text
2 Instanzen
     ↓
4 Instanzen
     ↓
2 Instanzen
```

Zusätzlich wurden Autoscale-Regeln konfiguriert:

- Minimum: `2`
- Maximum: `6`
- Default: `2`
- CPU > 70 % → Scale Out
- CPU < 30 % → Scale In

Nach erfolgreicher Validierung wurde das VM Scale Set aus Kostengründen vollständig gelöscht.

---

##  Azure App Service

Azure App Service wurde als PaaS-Gegenstück zur klassischen VM praktisch betrachtet.

Verwendet wurden:

- Windows App Service
- ASP.NET Framework 4.8
- Premium App Service Plan
- zusätzlicher `staging` Deployment Slot

### Architekturentscheidung

**ASP.NET Framework 4.8 → Windows App Service**

Ein Linux App Service Plan wurde verworfen, da ASP.NET Framework 4.8 die Windows-Plattform benötigt.

---

## 🔄 Deployment Slots

Für die Web App wurde zusätzlich zum Production Slot ein:

`staging`

Slot erstellt.

Dadurch wurden folgende Konzepte praktisch betrachtet:

- Production vs Staging
- Slot Swap
- Rollback
- Trennung zwischen Deployment und produktiver Umgebung

Ein eindeutiger A/B-Inhaltsnachweis zwischen beiden Slots wurde im Lab nicht durchgeführt und wird deshalb nicht als praktisch validierter Code-Swap dokumentiert.

Die App-Service-Ressourcen wurden anschließend aus Kostengründen gelöscht.

---

##  Architekturentscheidungen

### Zweite VM in `vnet-test`

**Entscheidung:** VM-B wird im zweiten VNet platziert.

**Warum:** Dadurch kann echter End-to-End-Traffic über das bestehende VNet Peering getestet werden.

---

### Keine Public IP

**Entscheidung:** VM-B erhält keine Public IP.

**Warum:** Die Workload benötigt ausschließlich interne Kommunikation und Administration über den privaten Netzwerkpfad.

---

### Separate Data Disk

**Entscheidung:** zusätzliche Daten werden nicht ausschließlich auf der OS Disk gespeichert.

**Warum:** Betriebssystem und Nutzdaten bleiben getrennt verwaltbar.

---

### Custom Script Extension statt manueller Konfiguration

**Entscheidung:** Post-Deployment-Konfiguration wird automatisiert ausgeführt.

**Warum:** Die Konfiguration ist reproduzierbarer als manuelle Änderungen über SSH.

---

### Uniform VMSS

**Entscheidung:** Uniform Orchestration für den homogenen Test-Workload.

**Warum:** Alle Instanzen verwenden dasselbe VM-Modell und werden gemeinsam skaliert.

---

### Temporäre Ressourcen löschen

VMSS, App Service Plan, Web App und Deployment Slot wurden nach erfolgreichem Lern- und Validierungseffekt wieder entfernt.

Damit entstehen keine unnötigen dauerhaften Compute-Kosten.

---

##  Offene Punkte

Nicht praktisch vollständig umgesetzt beziehungsweise validiert wurden:

- Proximity Placement Groups
- Availability Sets
- Snapshot-/Image-Szenarien
- Container Apps
- vollständiger App-Service-A/B-Code-Swap
- eigener Outbound-Internetpfad für VM-B

Ein NAT Gateway wurde bewusst nicht nur für die Installation eines Test-Webservers erstellt.

---

##  Nächster Schritt

Die beiden bestehenden Compute-Endpunkte werden im letzten Fachbereich **Monitor & Maintain** weiterverwendet.

Insbesondere eignen sich:

- `VM-A`
- `VM-B`
- VNet Peering
- getrennte NSGs
- `vm-b-data01`
- Private Storage-Anbindung
- vorhandene Log-Analytics-/Application-Insights-Ressourcen

für praktische Szenarien mit:

**Azure Monitor, Log Analytics, Alerts, Connection Monitor und Azure Backup.**
