# 📊 05 – Monitor & Maintain

Abschluss des Master Labs mit Fokus auf **Azure Monitor, Log Analytics, Alerting, Network Monitoring und Azure Backup**.

Die bereits vorhandenen Compute-, Netzwerk- und Storage-Ressourcen wurden weiterverwendet, um Monitoring und Recovery an realen Workloads zu validieren.

---

## Architektur

```text
VM-A
├── Azure Monitor Agent
│       ↓
├── Data Collection Rule
│       ↓
├── Log Analytics Workspace
│
└── CPU Metric Alert
        ↓
   Action Group
        ↓
   E-Mail Notification


VM-A ── Connection Monitor / TCP 22 ── VM-B


testostorageb
        ↓
Diagnostic Settings
        ↓
StorageBlobLogs
        ↓
Log Analytics / KQL


VM-B
 ↓
Azure Backup
 ↓
Recovery Services Vault
 ↓
Recovery Point
```

---

## Umgesetzt

- bestehender Log Analytics Workspace weiterverwendet
- Azure Monitor Agent auf `VM-A`
- Linux Data Collection Rule `dcr-az104-linux`
- Performance Counter und Syslog
- KQL-Abfragen in Log Analytics
- CPU Metric Alert
- Action Group mit E-Mail-Benachrichtigung
- Connection Monitor zwischen `VM-A` und `VM-B`
- kontrolliertes NSG-Troubleshooting
- Network Watcher `IP Flow Verify`
- Diagnostic Settings für Blob Storage
- Auswertung von `StorageBlobLogs`
- Azure Activity Log untersucht
- Recovery Services Vault
- Azure VM Backup für `VM-B`
- On-Demand Backup
- Recovery Point

---

## Azure Monitor Agent & Data Collection Rule

Für Guest-OS-Telemetrie wurde auf `VM-A` der:

`AzureMonitorLinuxAgent`

installiert.

Die Data Collection Rule:

`dcr-az104-linux`

ist mit `VM-A` und `VM-B` verbunden und verwendet den bestehenden Log Analytics Workspace als Destination.

### Gesammelte Daten

**Performance Counter**

- Processor Time
- Available Memory
- Used Memory

**Linux Syslog**

- `auth`
- `authpriv`
- `daemon`
- `syslog`

Minimum Log Level:

`Info`

Für `VM-A` wurde der vollständige Datenpfad praktisch validiert:

```text
VM-A
 ↓
Azure Monitor Agent
 ↓
dcr-az104-linux
 ↓
Log Analytics Workspace
 ↓
Heartbeat / Perf / Syslog
 ↓
KQL
```

---

## Metric Alert & Action Group

Für `VM-A` wurde der Metric Alert:

`alert-vma-cpu`

erstellt.

### Konfiguration

- Metric: `Percentage CPU`
- Aggregation: Average
- Threshold: `> 80 %`
- Window Size: 5 Minuten
- Evaluation Frequency: 1 Minute
- Severity: 2

Benachrichtigungen werden über:

`ag-az104-ops`

versendet.

Eine kontrollierte CPU-Last wurde auf `VM-A` erzeugt.

Der Alert wurde tatsächlich ausgelöst und die konfigurierte **E-Mail-Benachrichtigung** empfangen.

---

## Connection Monitor

Die bereits vorhandene private Verbindung zwischen den beiden VMs wurde mit:

`cm-vma-vmb`

überwacht.

```text
VM-A
 │
 │ TCP 22
 │
 ▼
VM-B
```

### Normalzustand

- Connectivity: erfolgreich
- Checks Failed: `0 %`
- RTT: ca. `1,27 ms`

Damit wurde nicht nur die Netzwerkverbindung getestet, sondern ein bereits vorhandener produktionsähnlicher Pfad kontinuierlich überwacht.

---

## Network Troubleshooting

Für einen kontrollierten Fehler wurde auf `nsg-vm-b` temporär die Regel:

`BlockVM-A`

erstellt.

Die Regel blockierte TCP 22 von `VM-A` zu `VM-B`.

### Ergebnis

Connection Monitor wechselte vom funktionierenden Zustand auf **Fail**.

Anschließend wurde mit **IP Flow Verify** geprüft:

```text
Direction: Inbound
Destination VM: VM-B
Local IP: 10.0.0.4
Local Port: 22
Remote IP: 172.16.0.4
Result: Access denied
```

Als Ursache wurden konkret identifiziert:

- NSG: `nsg-vm-b`
- Rule: `BlockVM-A`

Nach Entfernen der Regel wurde die Verbindung wieder erfolgreich.

### Erkenntnis

**Connection Monitor** zeigt, dass ein Netzwerkpfad fehlschlägt.

**IP Flow Verify** kann anschließend helfen, die konkrete blockierende NSG-Regel zu identifizieren.

---

## Diagnostic Settings & KQL

Für den Blob-Service von `testostorageb` wurde das Diagnostic Setting:

`diag-blob-law`

konfiguriert.

Erfasst werden:

- `StorageRead`
- `StorageWrite`
- `StorageDelete`

Die Daten werden an den vorhandenen Log Analytics Workspace gesendet.

### KQL

Die eingegangenen Storage Logs wurden unter anderem mit folgender Query untersucht:

```kusto
StorageBlobLogs
| where TimeGenerated > ago(30m)
| where AccountName == "testostorageb"
| project TimeGenerated, OperationName, StatusCode, StatusText, CallerIpAddress, Uri
| order by TimeGenerated desc
```

Beobachtete Operationen:

- `PutBlob`
- `DeleteBlob`
- `ListBlobs`
- `GetBlob`
- `GetBlobProperties`

Dabei wurden unter anderem erfolgreiche Requests sowie Fehler wie `404 BlobNotFound` sichtbar.

---

## Activity Log

Das Azure Activity Log wurde anhand aktueller Management-Operationen untersucht.

Dabei wurde zwischen verschiedenen Telemetriequellen unterschieden:

```text
Activity Log
→ Azure Control Plane

Guest OS Logs
→ Betriebssystem innerhalb der VM

Storage Resource Logs
→ Telemetrie der Azure Storage Resource
```

Damit werden Plattform-, Ressourcen- und Guest-OS-Daten nicht als identische Logquellen behandelt.

---

## Azure Backup

Für `VM-B` wurde der Recovery Services Vault:

`rsv-az104-weu`

erstellt.

### Konfiguration

- Region: West Europe
- Backup Storage Redundancy: LRS
- Soft Delete: 14 Tage
- Immutability: Disabled

`VM-B` wurde anschließend als Protected Item aufgenommen.

Praktisch validiert wurden:

- Backup Protection aktiviert
- On-Demand Backup gestartet
- Backup Job erfolgreich abgeschlossen
- Recovery Point vorhanden
- Restore VM, Restore Disks und File Recovery betrachtet

### Backup vs. Site Recovery

**Azure Backup**

→ Wiederherstellung eines früheren Daten- oder VM-Zustands über Recovery Points.

**Azure Site Recovery**

→ Replikation und Failover einer Workload für Disaster-Recovery-Szenarien.

---

## Architekturentscheidungen

### Bestehenden Log Analytics Workspace weiterverwenden

**Entscheidung:** vorhandenen Workspace nutzen.

**Warum:** Ein geeigneter Workspace in West Europe existierte bereits.

**Alternative:** neuer dedizierter Workspace.

**Warum nicht:** unnötige Duplizierung und Verteilung der Monitoring-Daten.

---

### Metric Alert für CPU

**Entscheidung:** CPU direkt über eine Azure-Plattformmetrik überwachen.

**Warum:** `Percentage CPU` steht ohne Guest Agent als Plattformmetrik zur Verfügung.

**Alternative:** CPU ausschließlich über Log Analytics.

**Warum nicht:** für einen einfachen Schwellenwert unnötig komplex.

---

### AMA + DCR für Guest-OS-Daten

**Entscheidung:** Azure Monitor Agent mit Data Collection Rule.

**Warum:** Syslog und Guest-OS-Performance-Daten benötigen einen Datensammlungsweg aus der VM.

**Alternative:** nur Plattformmetriken verwenden.

**Warum nicht:** Guest-OS-Telemetrie wäre damit nicht vollständig abgedeckt.

---

### Bestehenden Netzwerkpfad überwachen

**Entscheidung:** `VM-A → VM-B` über TCP 22.

**Warum:** Der Netzwerkpfad existierte bereits und wurde zuvor praktisch über VNet Peering validiert.

**Alternative:** künstliche Monitoring-Endpunkte erstellen.

**Warum nicht:** zusätzlicher Ressourcenaufwand ohne Mehrwert.

---

## Validierung

Erfolgreich praktisch nachgewiesen:

- Azure Monitor Agent auf `VM-A`
- AMA Provisioning State: `Succeeded`
- Heartbeat in Log Analytics
- Performance-Daten in `Perf`
- Syslog-Daten in `Syslog`
- Memory Performance Counter
- CPU Metric Alert
- E-Mail über Action Group
- Connection Monitor zwischen zwei echten VMs
- kontrollierter Netzwerkfehler
- Root Cause über IP Flow Verify
- Storage Diagnostic Logs
- KQL auf `StorageBlobLogs`
- Activity Log
- Azure VM Backup
- erfolgreicher On-Demand Backup Job
- vorhandener Recovery Point

---

## Einschränkungen

Nicht als vollständig praktisch validiert dokumentiert:

- AMA-Telemetrie von `VM-B`
- vollständige Storage Alert Rule Landschaft
- Backup Reports
- alte Application-Insights-/Smart-Detection-Ressourcen wurden bewusst nicht vollständig bereinigt

Diese Punkte waren für den Abschluss des zusammenhängenden Master Labs nicht erforderlich.

---

## Ergebnis

Mit Monitor & Maintain wurde die bestehende Azure-Architektur um eine zentrale Betriebs- und Recovery-Schicht erweitert.

Die Umgebung kann damit nicht nur bereitgestellt und verbunden, sondern auch:

**überwacht · analysiert · alarmiert · diagnostiziert · gesichert**

werden.

Damit sind alle fünf Core-Bereiche des **AZ-104 Master Labs** abgeschlossen.
