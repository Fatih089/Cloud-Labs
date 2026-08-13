# 🔐 01 – Identity & Governance

Grundlage der Azure-Umgebung mit Fokus auf **Azure RBAC, Managed Identities, Resource Locks und Azure Policy**.

Die in diesem Lab erstellten Ressourcen werden von den nachfolgenden Networking-, Storage-, Compute- und Monitoring-Bereichen weiterverwendet.

---

##  Umgesetzt

* Azure RBAC und Scope-Vererbung
* System-assigned Managed Identity für `VM-A`
* Resource Locks mit `CanNotDelete` und `ReadOnly`
* Azure Policy mit `Deny`
* Policy Exclusion auf Resource-Group-Ebene
* Resource-Move-Szenarien zwischen Resource Groups
* Unterscheidung zwischen Management Plane und Data Plane

---

##  Zentrale Ressourcen

| Ressource            | Typ             | Resource Group      | Region      |
| -------------------- | --------------- | ------------------- | ----------- |
| `VM-A`               | Virtual Machine | `RG-A`              | West Europe |
| `vnet-westeurope-1`  | Virtual Network | `RG-A`              | West Europe |
| `snet-westeurope-1`  | Subnet          | `RG-A`              | West Europe |
| `testostorageb`      | Storage Account | bestehende Umgebung | West Europe |
| `RG-Policy-Test`     | Resource Group  | —                   | West Europe |
| `RG-Policy-Ausnahme` | Resource Group  | —                   | West Europe |

`VM-A` verwendet eine **System-assigned Managed Identity** und bildet die zentrale Test- und Verwaltungsressource der Umgebung.

---

##  Architekturentscheidungen

### Least Privilege bei Azure RBAC

Berechtigungen werden nach dem Modell **Principal → Role → Scope** vergeben.

Rollen sollen möglichst auf dem kleinsten sinnvollen Scope zugewiesen werden, statt unnötig weitreichende Berechtigungen auf Subscription-Ebene zu vergeben.

### Managed Identity statt gespeicherter Secrets

`VM-A` verwendet eine System-assigned Managed Identity.

Damit kann die VM gegenüber Azure-Diensten authentifiziert werden, ohne Benutzerkennwörter oder langfristige Access Keys innerhalb der Workload speichern zu müssen.

### Resource Locks

Für unterschiedliche Schutzanforderungen wurden beide Lock-Typen praktisch getestet:

* `CanNotDelete` verhindert das Löschen, erlaubt aber Änderungen.
* `ReadOnly` verhindert sowohl Änderungen als auch Löschoperationen.

### Azure Policy statt Resource Lock

Eine Azure Policy mit `Deny` wurde verwendet, um die Erstellung von Virtual Networks auf einem definierten Scope zu verhindern.

Eine separate Resource Group wurde über eine **Policy Exclusion** ausgenommen.

Resource Locks wären für diese Anforderung ungeeignet, da sie vorhandene Ressourcen schützen, aber keine Governance-Regel für die Erstellung bestimmter Ressourcentypen darstellen.

---

##  Validierung

Folgende Szenarien wurden praktisch überprüft:

* `CanNotDelete` blockierte Löschoperationen, während Änderungen weiterhin möglich waren.
* `ReadOnly` blockierte Änderungen und Löschoperationen.
* Ein `ReadOnly` Lock auf Resource-Group-Ebene wirkte auf enthaltene Ressourcen.
* Resource Moves wurden in Verbindung mit unterschiedlichen Lock-Scopes getestet.
* Die `Deny` Policy blockierte die Erstellung eines Virtual Networks außerhalb der Exclusion.
* Innerhalb von `RG-Policy-Ausnahme` war die VNet-Erstellung möglich.
* Ein Resource Move in einen durch die Policy geschützten Scope wurde blockiert.
* Die System-assigned Managed Identity von `VM-A` wurde erfolgreich aktiviert.

---

##  Lab-Einschränkungen

Die Umgebung wird innerhalb eines bestehenden Azure-Tenants betrieben.

Aufgrund vorhandener Tenant- und ABAC-Einschränkungen konnten:

* kein separater Entra-Testbenutzer erstellt werden,
* einzelne RBAC-Zuweisungen an die Managed Identity nicht vollständig End-to-End validiert werden.

Nicht erfolgreich getestete Szenarien werden im Projekt nicht als erfolgreich umgesetzt dargestellt.

---

##  Weiterverwendung

Die vorhandene Basis wurde anschließend direkt im Networking-Lab weiterverwendet:

* `VM-A`
* `vnet-westeurope-1`
* `snet-westeurope-1`
* Network Interface von `VM-A`
* System-assigned Managed Identity
* `testostorageb`

Dadurch entsteht eine zusammenhängende Azure-Umgebung statt voneinander isolierter Einzel-Labs.
