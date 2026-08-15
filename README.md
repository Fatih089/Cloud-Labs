# ☁️ Azure AZ-104 Master Lab

Praxisorientiertes Azure-Infrastrukturprojekt mit Fokus auf **Administration, Networking, Security und Troubleshooting**.

Die Umgebung wird schrittweise als zusammenhängende Azure-Architektur in **West Europe** aufgebaut, validiert und erweitert.

---

##  Projektstatus

- ✅ Identity & Governance
- ✅ Networking
- ✅ Storage
- ✅ Compute
- 🔄 Monitoring

---

##  Architektur

```text
VM-A ───────── VNet Peering ───────── VM-B
 │                                      │
 │                                      ├── Data Disk
 │                                      └── Custom Script Extension
 │
 └── vnet-westeurope-1                  └── vnet-test
     ├── snet-westeurope-1                  10.0.0.0/16
     └── snet-app-westeurope-1
              │
              ├── Blob Private Endpoint
              └── File Private Endpoint
                        │
                        ▼
                  testostorageb
```

---

##  Azure Skills

`Azure RBAC` · `Managed Identity` · `Azure Policy` · `Resource Locks`  
`VNet Peering` · `NSG` · `ASG` · `Private DNS` · `Network Watcher`  
`Blob Storage` · `Azure Files` · `Private Endpoint` · `Lifecycle Management`  
`Virtual Machines` · `Managed Disks` · `VM Extensions` · `VM Scale Sets`  
`App Service` · `Deployment Slots`

---

##  Labs

- [01 – Identity & Governance](./01-Identity-Governance/)
- [02 – Networking](./02-Networking/)
- [03 – Storage](./03-Storage/)
- [04 – Compute](./04-Compute/) 
- **05 – Monitoring** 🔄

---

##  Ziel

Aufbau einer Azure-Umgebung, die ich **bereitstellen, absichern, validieren und systematisch troubleshootieren** kann.

Nach Monitoring folgen **End-to-End-Szenarien, gezieltes Schließen offener Architektur-Lücken und Infrastructure as Code**.
