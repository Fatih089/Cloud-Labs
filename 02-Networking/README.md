# 🌐 02 – Networking

Erweiterung der bestehenden Azure-Umgebung um **Netzwerksegmentierung, VNet Peering, Network Security, Private DNS und Routing-Troubleshooting**.

Die vorhandene Infrastruktur aus Identity & Governance wurde weiterverwendet und später durch Compute, Storage und Monitoring End-to-End validiert.

---

## Netzwerkarchitektur

### `vnet-westeurope-1`

- Address Space: `172.16.0.0/16`
- `snet-westeurope-1` → `172.16.0.0/24`
- `snet-app-westeurope-1` → `172.16.1.0/24`

### `vnet-test`

- Address Space: `10.0.0.0/16`
- `default` → `10.0.0.0/24`

Beide Virtual Networks befinden sich in **West Europe** und sind bidirektional über **VNet Peering** verbunden.

```text
VM-A                              VM-B
172.16.0.x                        10.0.0.4
    │                                │
    ▼                                ▼
vnet-westeurope-1 ── VNet Peering ── vnet-test
    │
    ├── snet-westeurope-1
    │
    └── snet-app-westeurope-1
             │
             ├── Blob Private Endpoint
             └── File Private Endpoint
```

---

## Umgesetzt

- Segmentierung des bestehenden VNets über mehrere Subnets
- bidirektionales VNet Peering
- Network Security Groups auf Subnet- und NIC-Ebene
- Application Security Group für rollenbasierte NSG-Regeln
- Private DNS Zone `contoso.internal`
- VNet Links mit unterschiedlicher Auto-registration-Konfiguration
- DNS Auto-registration für `VM-A`
- Effective Routes
- Network Watcher
- `IP Flow Verify`
- `Next Hop`
- kontrolliertes NSG-Troubleshooting
- kontrolliertes Routing-Troubleshooting mit User Defined Route
- echter VM-zu-VM-Traffic über VNet Peering
- Connection Monitor zwischen `VM-A` und `VM-B`

---

## Network Security

Für `VM-A` wurden Security Controls auf zwei Ebenen eingesetzt:

- Subnet-NSG: `nsg-snet-westeurope-1`
- NIC-NSG: `VM-A-nsg`

Damit wurde praktisch validiert, dass ein Netzwerkfluss bei mehreren angewendeten NSGs die relevanten Regeln auf **beiden Ebenen** erfüllen muss.

### Application Security Group

`VM-A` wurde der Application Security Group:

`asg-app-westeurope`

zugeordnet.

Eine NSG-Regel erlaubt HTTP-Traffic aus `10.0.0.0/16` gezielt zur ASG, ohne einzelne private IP-Adressen dauerhaft in rollenbezogenen Security Rules pflegen zu müssen.

Dadurch können Server anhand ihrer logischen Rolle gruppiert werden.

---

## VNet Peering

`vnet-westeurope-1` und `vnet-test` wurden über bidirektionales **VNet Peering** verbunden.

### Architekturentscheidung

**Entscheidung:** VNet Peering statt VPN Gateway

**Warum:** Beide Netzwerke befinden sich in Azure und benötigen eine direkte private Verbindung.

**Alternative:** VPN Gateway

**Warum nicht:** Für dieses Szenario unnötige Komplexität und zusätzliche laufende Kosten.

Azure stellt für das Peering automatisch entsprechende System Routes bereit. Eine permanente User Defined Route zum Peer-VNet ist daher nicht erforderlich.

---

## End-to-End-Validierung

Im späteren Compute-Lab wurde mit `VM-B` ein echter Endpunkt in `vnet-test` bereitgestellt.

Anschließend wurde eine private SSH-Verbindung getestet:

```text
VM-A
172.16.0.x
    │
    │ TCP 22
    │ VNet Peering
    ▼
VM-B
10.0.0.4
```

Die Verbindung war erfolgreich.

Damit wurden praktisch validiert:

- private VM-zu-VM-Kommunikation
- VNet Peering
- Azure Routing
- NSG-Verhalten
- tatsächlicher End-to-End-Traffic zwischen beiden VNets

---

## Private DNS

Private DNS Zone:

`contoso.internal`

### VNet Links

- `vnet-westeurope-1` → Auto-registration aktiviert
- `vnet-test` → Auto-registration deaktiviert

Für `VM-A` wurde automatisch folgender Record registriert:

```text
vm-a.contoso.internal → 172.16.0.4
```

Die interne Namensauflösung wurde mit `nslookup` erfolgreich validiert.

Im späteren Storage- und Compute-Lab wurde die DNS-Architektur zusätzlich um dienstspezifische Private-Link-Zonen erweitert.

---

## Routing-Troubleshooting

Für einen kontrollierten Fehler wurde temporär folgende User Defined Route erstellt:

```text
Destination: 10.0.0.0/16
Next Hop Type: None
```

`Effective Routes` zeigte die User Defined Route anschließend als aktiven Pfad.

Network Watcher **Next Hop** bestätigte, dass Traffic zum Peer-VNet nicht mehr über den vorgesehenen Peering-Pfad weitergeleitet wurde.

Nach Entfernen der fehlerhaften Route wurde wieder die automatisch von Azure bereitgestellte **VNet-Peering-Route** verwendet.

### Erkenntnis

User Defined Routes können Azure System Routes überschreiben.  
`Effective Routes` und `Next Hop` ermöglichen eine gezielte Analyse des tatsächlich verwendeten Routing-Pfads.

---

## NSG-Troubleshooting

Im späteren Compute- und Monitoring-Lab wurde der reale Pfad:

`VM-A → VM-B → TCP 22`

für ein kontrolliertes Fehlerszenario verwendet.

Eine temporäre Deny-Regel auf `nsg-vm-b` blockierte den SSH-Traffic.

### Analyse

**Connection Monitor**

- zeigte den Ausfall des Netzwerkpfads

**IP Flow Verify**

- Ergebnis: `Access denied`
- verursachende NSG: `nsg-vm-b`
- verursachende Regel: `BlockVM-A`

Nach Entfernen der Regel war die Verbindung wieder erfolgreich.

Im Normalzustand zeigte Connection Monitor:

- Checks Failed: `0 %`
- RTT: ca. `1,27 ms`

---

## Weiterverwendung im Master Lab

Die Netzwerkarchitektur wurde von den folgenden Bereichen weiterverwendet:

### Storage

- Private Endpoints für Blob Storage und Azure Files
- Nutzung von `snet-app-westeurope-1`
- Private DNS für Azure Private Link

### Compute

- `VM-B` als zweiter Endpunkt in `vnet-test`
- echter Traffic über VNet Peering
- kontrolliertes NSG-Troubleshooting

### Monitor & Maintain

- Connection Monitor zwischen `VM-A` und `VM-B`
- RTT- und Connectivity-Messung
- Root-Cause-Analyse mit IP Flow Verify

---

## Ergebnis

Die Networking-Schicht bildet die gemeinsame Connectivity-Basis des gesamten Master Labs.

Praktisch umgesetzt und validiert wurden:

**Segmentierung · Peering · Security · DNS · Routing · Private Connectivity · Monitoring · Troubleshooting**

Die Netzwerkarchitektur wurde nicht isoliert betrachtet, sondern anschließend von Storage-, Compute- und Monitoring-Workloads produktionsähnlich weiterverwendet.
