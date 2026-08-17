# ☁️  Azure Infrastructure Lab

Praxisorientiertes Azure-Infrastrukturprojekt mit Fokus auf **Administration, Networking, Security, Monitoring und Troubleshooting**.

Aufgebaut wurde eine zusammenhängende Azure-Umgebung in **West Europe**, in der die einzelnen Services bewusst miteinander verbunden und praktisch validiert wurden.

---

## Projektbereiche

- [01 – Identity & Governance](./01-Identity-Governance/)
- [02 – Networking](./02-Networking/)
- [03 – Storage](./03-Storage/)
- [04 – Compute](./04-Compute/)
- [05 – Monitor & Maintain](./05-Monitoring/)

---

## Architektur

```text
VM-A                              VM-B
 │                                  │
 ▼                                  ▼
vnet-westeurope-1 ── VNet Peering ── vnet-test
 │
 ├── snet-westeurope-1
 │
 └── snet-app-westeurope-1
          │
          ├── Blob Private Endpoint
          └── File Private Endpoint
                    │
                    ▼
              testostorageb

Azure Monitor
├── AMA + DCR
├── Log Analytics
├── Alerts
└── Connection Monitor
```

---

## Azure Skills

`Azure RBAC` · `Managed Identity` · `Azure Policy`  
`VNet Peering` · `NSG` · `ASG` · `Private DNS` · `Network Watcher`  
`Blob Storage` · `Azure Files` · `Private Endpoints`  
`Virtual Machines` · `Managed Disks` · `VM Extensions` · `VM Scale Sets`  
`Azure Monitor` · `Log Analytics` · `KQL` · `Azure Backup`

---

## Projektfokus
Nicht nur Ressourcen bereitstellen, sondern Azure-Infrastruktur technisch begründen, absichern, überwachen, validieren und systematisch troubleshootieren.
