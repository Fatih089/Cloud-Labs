# 💾 03 – Storage

Erweiterung der bestehenden Azure-Umgebung um **Blob Storage, Azure Files, Lifecycle Management, Private Endpoints und Private DNS**.

Der Fokus lag auf sicherem privaten Storage-Zugriff, Kostenoptimierung und der praktischen Abgrenzung zwischen Netzwerkzugriff und Datenberechtigungen.

---

##  Architektur

```text
VM-A
│
├── Blob Storage
│   └── Private DNS
│       └── pe-testostorageb-blob
│           └── 172.16.1.4
│
└── Azure Files
    └── Private DNS
        └── pe-testostorageb-file
            └── 172.16.1.5
                │
                ▼
          testostorageb
          StorageV2 / LRS
```

Beide **Private Endpoints** befinden sich im bestehenden:

`vnet-westeurope-1`
└── `snet-app-westeurope-1 (172.16.1.0/24)`

---

##  Umgesetzt

* Storage Account `testostorageb` als **General-purpose v2 (StorageV2)**
* Standard Performance mit **LRS**
* Default Access Tier: **Hot**
* privater Blob Container `media`
* Lifecycle Management Policy
* Azure File Share `appshare`
* separater Private Endpoint für `blob`
* separater Private Endpoint für `file`
* dienstspezifische Private DNS Zones
* AzCopy auf `VM-A`
* rekursiver Verzeichnis-Upload
* SMB-Zugriff auf Azure Files über Private Link
* Storage Firewall / Public Network Access getestet

---

##  Storage Account

### `testostorageb`

| Einstellung            | Konfiguration                  |
| ---------------------- | ------------------------------ |
| Account Kind           | StorageV2 / General-purpose v2 |
| Performance            | Standard                       |
| Redundancy             | LRS                            |
| Access Tier            | Hot                            |
| Secure Transfer        | Enabled                        |
| Minimum TLS            | 1.2                            |
| Anonymous Blob Access  | Disabled                       |
| Blob Soft Delete       | 7 Tage                         |
| Container Soft Delete  | 7 Tage                         |
| File Share Soft Delete | 7 Tage                         |

Der ursprünglich vorhandene `FileStorage`-Account wurde durch einen **StorageV2 Account** ersetzt, damit Blob Storage und Azure Files innerhalb eines gemeinsamen Storage Accounts verwendet werden können.

---

##  Private Storage Access

Für Blob Storage und Azure Files wurden getrennte Private Endpoints verwendet.

### Blob

```text
testostorageb
└── blob
    └── pe-testostorageb-blob
        └── 172.16.1.4
```

Private DNS Zone:

`privatelink.blob.core.windows.net`

### Azure Files

```text
testostorageb
└── file
    └── pe-testostorageb-file
        └── 172.16.1.5
```

Private DNS Zone:

`privatelink.file.core.windows.net`

Damit bleibt die vorhandene interne Zone `contoso.internal` für eigene interne Namen reserviert und wird nicht für Azure Private Link zweckentfremdet.

---

##  DNS-Validierung

Die private Namensauflösung wurde direkt von `VM-A` getestet.

### Blob Endpoint

```text
testostorageb.blob.core.windows.net
        ↓ CNAME
testostorageb.privatelink.blob.core.windows.net
        ↓
172.16.1.4
```

### File Endpoint

```text
testostorageb.file.core.windows.net
        ↓ CNAME
testostorageb.privatelink.file.core.windows.net
        ↓
172.16.1.5
```

Damit wurde bestätigt, dass Storage-Traffic innerhalb des VNets auf die privaten Endpunkte aufgelöst wird.

---

##  Lifecycle Management

Für den Blob Container `media` wurde die Policy:

`media-lifecycle`

konfiguriert.

```text
Hot
 ↓ nach 30 Tagen
Cool
 ↓ nach 180 Tagen
Delete
```

Die Regel gilt für Blob-Daten unter:

`media/`

Ziel ist die automatische Reduzierung langfristiger Speicherkosten und die Bereinigung alter Testdaten.

---

##  Blob Storage vs. Azure Files

### Blob Storage

Verwendet für objektbasierte Anwendungs- und Mediendaten.

Container:

`media`

### Azure Files

Verwendet als klassische SMB-Dateifreigabe.

File Share:

`appshare`

`appshare` wurde auf `VM-A` über **SMB 3.1.1** eingebunden.

Der Mount verwendete dabei den privaten File Endpoint:

`172.16.1.5`

Anschließend wurde erfolgreich eine Datei von `VM-A` auf dem Share erstellt.

---

##  AzCopy

AzCopy wurde auf `VM-A` installiert und für einen Verzeichnis-Upload verwendet.

Ein erster Upload ohne:

`--recursive`

scheiterte erwartungsgemäß, da eine Verzeichnisstruktur übertragen werden sollte.

Der anschließende rekursive Upload war erfolgreich:

```text
Transfers: 4
Successful: 4
Failed: 0
Status: Completed
```

Übertragen wurden unter anderem:

```text
AppMediaData/document.docx
AppMediaData/video.mp4
AppMediaData/Images/image1.jpg
AppMediaData/Images/image2.jpg
```

---

##  Troubleshooting

### Storage Firewall vs. Berechtigung

Bei deaktiviertem beziehungsweise eingeschränktem **Public Network Access** wurde der Zugriff vom lokalen Client durch die Storage-Netzwerkebene blockiert.

Damit wurde praktisch zwischen zwei getrennten Zugriffsebenen unterschieden:

```text
Netzwerkzugriff
      +
Authentifizierung / Berechtigung
      =
erfolgreicher Datenzugriff
```

Eine korrekte RBAC-Rolle allein reicht nicht aus, wenn die Storage Firewall den Netzwerkpfad blockiert.

---

### Private Endpoint erreichbar, anonymer Zugriff blockiert

Ein `curl`-Test von `VM-A` erreichte Blob Storage über den privaten Netzwerkpfad.

Die Antwort:

`409 Public access is not permitted on this storage account`

zeigte, dass **DNS und Netzwerkpfad funktionierten**, während anonymer Datenzugriff weiterhin verhindert wurde.

---

##  Architekturentscheidungen

### StorageV2 statt FileStorage

**Entscheidung:** General-purpose v2

**Warum:** Blob Storage, Azure Files, Access Tiers und Lifecycle Management werden gemeinsam benötigt.

---

### LRS für die Lab-Umgebung

**Entscheidung:** Locally Redundant Storage

**Warum:** Für Testdaten besteht kein Bedarf an zusätzlicher Zone- oder Geo-Redundanz. Dadurch bleiben die laufenden Kosten niedrig.

---

### Separate Private Endpoints

**Entscheidung:** Eigener Private Endpoint für `blob` und `file`.

**Warum:** Azure Storage Private Link arbeitet mit separaten Storage-Subresources.

---

### Private Endpoints im Application Subnet

**Entscheidung:** `snet-app-westeurope-1`

**Warum:** Private Service Endpoints bleiben logisch von der bestehenden VM-Workload getrennt.

---

### Least Privilege für Blob-Zugriff

Für die Managed Identity von `VM-A` wurde:

`Storage Blob Data Reader`

auf Container-Scope `media` als passende Rolle vorgesehen.

Die praktische Rollenzuweisung konnte aufgrund bestehender Tenant-/ABAC-Einschränkungen jedoch nicht erfolgreich abgeschlossen werden.

---

##  Einschränkungen

Folgende Punkte wurden nicht als erfolgreich umgesetzt dokumentiert:

* Managed Identity + Storage RBAC konnte wegen ABAC nicht End-to-End validiert werden.
* Azure CLI Device-Code-Login wurde durch Conditional Access blockiert.
* Azure File Sync wurde nicht praktisch aufgebaut.
* kein praktischer Wechsel zwischen LRS, ZRS oder Geo-Redundanz durchgeführt.
* Storage Service Endpoints wurden nicht zusätzlich aufgebaut.
* der finale Zustand von `Public Network Access` muss nach einer temporären Client-IP-Freigabe nochmals geprüft werden.

---

##  Nächster Schritt

Die Storage-Architektur wird im nächsten Abschnitt von **Compute** weiterverwendet.

Dabei bleiben insbesondere erhalten:

* `testostorageb`
* `media`
* `appshare`
* beide Private Endpoints
* beide Private DNS Zones
* `vnet-westeurope-1`
* `snet-app-westeurope-1`
* `VM-A`

Damit kann die nächste Compute-Workload direkt auf die bestehende private Netzwerk- und Storage-Architektur aufbauen.
