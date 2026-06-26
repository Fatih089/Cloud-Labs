# Lab N1 – SNAT Exhaustion: NAT Gateway vs Load Balancer

**Dauer:** 60–90 Minuten  
**Ziel:** Verstehen, wie SNAT-Ports entstehen, warum ein Standard Load Balancer an Grenzen stoßen kann und wie ein Azure NAT Gateway SNAT-Exhaustion verhindert.

---

# 1. Umgebung vorbereiten

## Resource Group erstellen

Portal → **Resource groups** → **Create**

- **Name:** `rg-n1-snat`
- **Region:** `West Europe`

Create.

---

## Virtual Network erstellen

Portal → **Virtual networks** → **Create**

- **Name:** `vnet-n1`
- **Address space:** `10.10.0.0/16`

Subnet erstellen:

- **Subnet name:** `vm-subnet`
- **Address range:** `10.10.1.0/24`

Create.

---

# 2. VM Scale Set ohne NAT Gateway deployen

Portal → **Virtual machine scale sets** → **Create**

## Basics

- **Name:** `vmss-n1`
- **Image:** Ubuntu LTS
- **Instances:** 2–3
- **Authentication:** SSH key oder Password

## Networking

- **Virtual Network:** `vnet-n1`
- **Subnet:** `vm-subnet`
- **Public IP:** None
- **Load Balancer:** Standard Load Balancer (auto-create)

Azure stellt automatisch **Outbound Connectivity über Load Balancer SNAT** bereit.

Create.

---

# 3. Outbound Traffic erzeugen

Verbinde dich mit einer VMSS-Instanz über **Azure Bastion** oder eine **Jumpbox**.

Führe folgenden Befehl aus, um viele kurzlebige Outbound-Verbindungen zu erzeugen.

```bash
for i in {1..20000}; do curl -m 1 https://microsoft.com >/dev/null 2>&1 & done
```

Alternativ stabiler:

```bash
seq 1 20000 | xargs -n1 -P200 curl -m 1 https://microsoft.com >/dev/null 2>&1
```

---

# 4. SNAT Verhalten beobachten

Der **Standard Load Balancer** weist jedem Backend-Host eine begrenzte Anzahl an **SNAT-Ports** zu.

Wenn viele Outbound-Verbindungen gleichzeitig erstellt werden:

- Einige Verbindungen können fehlschlagen
- Timeouts können auftreten
- SNAT-Ports können erschöpft sein

Du kannst auch die verwendete Public IP prüfen:

```bash
curl ifconfig.me
```

Erwartetes Ergebnis:

Die **Frontend Public IP des Load Balancers** wird als Source IP verwendet.

---

# 5. NAT Gateway deployen

## Public IP erstellen

Portal → **Public IP addresses** → **Create**

- **Name:** `pip-nat`
- **SKU:** Standard
- **Assignment:** Static
- **Resource group:** `rg-n1-snat`

Create.

---

## NAT Gateway erstellen

Portal → **NAT gateways** → **Create**

- **Name:** `nat-n1`
- **Public IP configuration:** Add existing → `pip-nat`
- **Idle timeout:** 4 minutes (default)

Create.

---

## NAT Gateway mit Subnet verknüpfen

Portal → **Virtual networks** → `vnet-n1`

Dann:

**Subnets → vm-subnet**

Konfiguration:

- **NAT Gateway:** `nat-n1`

Save.

Wichtig:

Ein NAT Gateway wirkt auf **Subnet-Ebene**.

---

# 6. Lösung validieren

Führe den Test erneut aus:

```bash
for i in {1..20000}; do curl -m 1 https://microsoft.com >/dev/null 2>&1 & done
```

Überprüfe erneut die Public IP:

```bash
curl ifconfig.me
```

Erwartetes Ergebnis:

- Die Outbound IP sollte jetzt **pip-nat** sein
- Das NAT Gateway stellt einen **deutlich größeren SNAT-Port-Pool** bereit
- Verbindungsfehler sollten deutlich seltener auftreten

---

# 7. Wichtigste Erkenntnisse

**Standard Load Balancer**

- Begrenzte Anzahl an SNAT-Ports pro Backend-Instanz
- Kann bei hoher Outbound-Last zu **SNAT Exhaustion** führen

**NAT Gateway**

- Entwickelt für **skalierbare Outbound Connectivity**
- Stellt **64.000 SNAT-Ports pro Public IP** bereit
- Empfohlene Lösung für Workloads mit vielen Outbound-Verbindungen

---

# 8. Cleanup

Lösche alle Ressourcen, um Kosten zu vermeiden.

```bash
az group delete --name rg-n1-snat --yes --no-wait
```
