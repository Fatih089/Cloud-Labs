# ☁️ Azure AZ-104 Master Lab

**Praxisorientiertes Azure-Infrastrukturprojekt mit Fokus auf Azure Administration, Networking, Security und Troubleshooting.**

Aufbau einer zusammenhängenden Azure-Umgebung in **West Europe**.
Die einzelnen Bereiche bauen aufeinander auf und werden durch praktische Validierung und kontrollierte Fehlerszenarien ergänzt.

---

## 🚀 Projektstatus

* ✅ **Identity & Governance**
* ✅ **Networking**
* 🔄 **Storage**
* ⏳ **Compute**
* ⏳ **Monitoring**

---

## 🛠️ Tech Stack

`Azure RBAC` · `Managed Identity` · `Azure Policy` · `Resource Locks` · `Virtual Networks` · `VNet Peering` · `NSG` · `ASG` · `Private DNS` · `Network Watcher`

---

## 🏗️ Aktuelle Architektur

```text
VM-A
│
├── System-assigned Managed Identity
│
└── NIC
    ├── NSG
    ├── ASG
    │
    └── snet-westeurope-1
        │
        ▼
    vnet-westeurope-1
    172.16.0.0/16
        │
        │ VNet Peering
        ▼
    vnet-test
    10.0.0.0/16

Private DNS
└── contoso.internal

Storage
└── testostorageb
```

---

## 🔧 Bisher umgesetzt

### Identity & Governance

* Azure RBAC und Scope-Modell
* System-assigned Managed Identity
* Resource Locks
* Azure Policy und Policy Exclusions
* Resource-Move-Szenarien

### Networking

* VNet- und Subnet-Segmentierung
* Bidirektionales VNet Peering
* Network Security Groups auf Subnet- und NIC-Ebene
* Application Security Group
* Private DNS mit Auto-registration
* Effective Routes und User Defined Routes
* Network Watcher mit `IP Flow Verify` und `Next Hop`
* Kontrolliertes NSG- und Routing-Troubleshooting

---

## 🔍 Troubleshooting

Neben dem Deployment werden bewusst Fehlerszenarien eingebaut und systematisch analysiert.

**Beispiel:**

Eine temporäre **User Defined Route** leitete den Traffic für `10.0.0.0/16` auf den Next Hop `None`.

Die Ursache wurde mit:

* **Effective Routes**
* **Network Watcher – Next Hop**

identifiziert.

Nach Entfernen der fehlerhaften Route wurde wieder die automatisch von Azure bereitgestellte **VNet-Peering-Route** verwendet.

---

## 📂 Labs

* [01 – Identity & Governance](./01-Identity-Governance/)
* [02 – Networking](./02-Networking/)
* **03 – Storage** 🔄
* **04 – Compute** ⏳
* **05 – Monitoring** ⏳

---

## 🎯 Projektziel

Eine Azure-Umgebung aufzubauen, die ich nicht nur bereitstellen, sondern auch **architektonisch begründen, absichern, validieren und systematisch troubleshootieren** kann.

Nach Abschluss der Kernumgebung wird das Projekt um **End-to-End-Szenarien** und **Infrastructure as Code** erweitert.
