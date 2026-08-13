# 🌐 02 – Networking

Erweiterung der bestehenden Azure-Umgebung um **Netzwerksegmentierung, VNet Peering, Network Security, Private DNS und Routing-Troubleshooting**.

Die vorhandene Infrastruktur aus Identity & Governance wurde bewusst weiterverwendet und erweitert.

---

##  Netzwerkarchitektur

### `vnet-westeurope-1`

* Address Space: `172.16.0.0/16`
* `snet-westeurope-1` → `172.16.0.0/24`
* `snet-app-westeurope-1` → `172.16.1.0/24`

### `vnet-test`

* Address Space: `10.0.0.0/16`
* `default` → `10.0.0.0/24`

Beide Virtual Networks befinden sich in **West Europe** und sind bidirektional über **VNet Peering** verbunden.

---

##  Umgesetzt

* Erweiterung des bestehenden VNets um ein separates Application Subnet
* Bidirektionales VNet Peering
* Network Security Groups auf Subnet- und NIC-Ebene
* Application Security Group für rollenbasierte NSG-Regeln
* Private DNS Zone `contoso.internal`
* VNet Links mit unterschiedlicher Auto-registration-Konfiguration
* DNS Auto-registration für `VM-A`
* Effective Routes
* Network Watcher
* `IP Flow Verify`
* `Next Hop`
* kontrolliertes NSG-Troubleshooting
* kontrolliertes Routing-Troubleshooting mit User Defined Route

---

##  Network Security

Für `VM-A` wurden Security Controls auf zwei Ebenen betrachtet:

* Subnet-NSG: `nsg-snet-westeurope-1`
* NIC-NSG: `VM-A-nsg`

Damit wurde praktisch überprüft, dass ein Netzwerkfluss bei mehreren angewendeten NSGs die relevanten Regeln auf **beiden Ebenen** erfüllen muss.

### Application Security Group

`VM-A` wurde der ASG:

`asg-app-westeurope`

zugeordnet.

Eine NSG-Regel erlaubt HTTP-Traffic aus `10.0.0.0/16` gezielt zur ASG, ohne die private IP-Adresse der VM dauerhaft als Ziel definieren zu müssen.

Dadurch können zukünftige Server anhand ihrer logischen Rolle gruppiert werden.

---

##  VNet Peering

`vnet-westeurope-1` und `vnet-test` wurden über bidirektionales **VNet Peering** verbunden.

Für das aktuelle Szenario wurde bewusst kein VPN Gateway verwendet.

### Entscheidung

**VNet Peering statt VPN Gateway**

**Grund:** Beide Netzwerke befinden sich in Azure und benötigen eine direkte private Verbindung. Ein VPN Gateway würde zusätzliche Komplexität und laufende Kosten erzeugen, ohne für dieses Szenario einen entsprechenden Mehrwert zu bieten.

Azure stellt die notwendigen Peering-Routen automatisch bereit. Eine permanente User Defined Route zum Peer-VNet ist daher nicht erforderlich.

---

##  Private DNS

Private DNS Zone:

`contoso.internal`

### VNet Links

* `vnet-westeurope-1` → Auto-registration aktiviert
* `vnet-test` → Auto-registration deaktiviert

Durch Auto-registration wurde für `VM-A` automatisch folgender Record erzeugt:

`vm-a.contoso.internal → 172.16.0.4`

Die Namensauflösung wurde innerhalb von `VM-A` erfolgreich mit `nslookup` validiert.

---

##  Troubleshooting

### NSG-Fehlerszenario

Bei einem SSH-Test erlaubte zunächst nur die Subnet-NSG TCP Port 22.

Die NIC-NSG fiel weiterhin auf die standardmäßige `DenyAllInBound`-Regel zurück.

**Ergebnis:** Die Verbindung wurde blockiert.

Nach einer passenden Allow-Regel auf beiden relevanten Ebenen funktionierte die Verbindung.

Die temporären SSH-Regeln wurden anschließend wieder entfernt.

---

### ASG-Validierung

Die Wirkung der Application Security Group wurde mit **Network Watcher – IP Flow Verify** geprüft.

**VM-A Mitglied der ASG:**
HTTP-Traffic wurde durch die ASG-basierte Allow-Regel erlaubt.

**VM-A aus der ASG entfernt:**
Der gleiche Traffic wurde durch die nachgelagerte Deny-Regel blockiert.

**VM-A erneut hinzugefügt:**
Die Allow-Regel griff wieder.

Damit wurde die Abhängigkeit zwischen ASG-Mitgliedschaft und NSG-Regel praktisch validiert.

---

### Routing-Fehlerszenario

Temporär wurde folgende User Defined Route erstellt:

**Destination:** `10.0.0.0/16`
**Next Hop Type:** `None`

`Effective Routes` zeigte anschließend die User Defined Route als aktive Route.

Network Watcher **Next Hop** bestätigte, dass Traffic zum Peer-VNet nicht mehr den vorgesehenen Peering-Pfad verwendete.

Nach Entfernen der temporären Route zeigte Azure wieder:

`VNet peering`

als effektiven Routing-Pfad.

Die temporäre Route Table wurde nach dem Troubleshooting vollständig entfernt.

---

##  Validierung

Praktisch geprüft wurden:

* nicht überlappende VNet Address Spaces
* bidirektionales VNet Peering
* automatisch erzeugte Peering-Route
* kombinierte Wirkung von Subnet- und NIC-NSGs
* ASG-basierte NSG-Regeln
* Private DNS Auto-registration
* `vm-a.contoso.internal → 172.16.0.4`
* DNS-Auflösung mit `nslookup`
* Network Watcher `IP Flow Verify`
* Network Watcher `Next Hop`
* `Effective Routes`
* Verhalten einer fehlerhaften User Defined Route
* Wiederherstellung des korrekten Peering-Routings

---

##  Noch nicht umgesetzt

Ein echter VM-zu-VM-Traffic zwischen beiden VNets wurde noch nicht durchgeführt, da im `vnet-test` keine zusätzliche VM bereitgestellt wurde.

Weitere Netzwerkkomponenten wie:

* VPN Gateway / Gateway Transit
* Azure Bastion
* Application Gateway
* Load Balancer
* Connection Monitor

werden nur dann ergänzt, wenn sie im weiteren Master Lab einen sinnvollen End-to-End-Anwendungsfall erhalten.

---

## Nächste Schritte

Die bestehende Netzwerkarchitektur wird im Storage-Lab um Private Endpoint und Private DNS erweitert.
