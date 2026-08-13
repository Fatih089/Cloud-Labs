# ☁️ Azure AZ-104 Master Lab

Praxisorientiertes Azure-Infrastrukturprojekt mit Fokus auf **Administration, Networking, Security und Troubleshooting**.

Die Umgebung wird schrittweise als zusammenhängende Azure-Architektur in **West Europe** aufgebaut, validiert und erweitert.

---

## 🚀 Projektstatus

* ✅ Identity & Governance
* ✅ Networking
* 🔄 Storage
* ⏳ Compute
* ⏳ Monitoring

---

## 🏗️ Architektur

```text
VM-A
└── vnet-westeurope-1 (172.16.0.0/16)
    ├── snet-westeurope-1
    ├── snet-app-westeurope-1
    └── VNet Peering
        └── vnet-test (10.0.0.0/16)

Private DNS: contoso.internal
Storage: testostorageb
```

---

## 🛠️ Azure Skills

`Azure RBAC` · `Managed Identity` · `Azure Policy` · `Resource Locks`
`VNet Peering` · `NSG` · `ASG` · `Private DNS` · `Network Watcher`

---

## 📂 Labs

* [01 – Identity & Governance](./01-Identity-Governance/)
* [02 – Networking](./02-Networking/)
* **03 – Storage** 🔄
* **04 – Compute** ⏳
* **05 – Monitoring** ⏳

---

## 🎯 Ziel

Aufbau einer Azure-Umgebung, die ich **bereitstellen, absichern, validieren und systematisch troubleshootieren** kann.

Nach Abschluss folgen **End-to-End-Szenarien** und **Infrastructure as Code**.
