# ☁️ Azure Shared Responsibility — Real-World Analysis
### Mapping Personal Home Lab Infrastructure to the Cloud Shared Responsibility Model

![AZ-900](https://img.shields.io/badge/AZ--900-Cloud_Concepts-0078D4?style=for-the-badge&logo=microsoftazure&logoColor=white)
![Model](https://img.shields.io/badge/Coverage-On--Prem_·_IaaS_·_PaaS-6C757D?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Complete-2ea44f?style=for-the-badge)

---

## The Business Problem

One of the most common and costly cloud security failures is **misplaced responsibility**. An organization assumes Microsoft is securing something that is actually their own responsibility, or they spend time and money securing something Microsoft already handles by default.

This gap causes three real business problems:

- **Compliance failures** — FedRAMP, NIST 800-53, and CMMC auditors require documented responsibility assignment for every security control
- **Security gaps** — assuming Microsoft patches the OS on an Azure VM (they don't. IaaS means you own the OS) leaves systems unpatched and vulnerable
- **Wasted security spend** — building controls around things Microsoft already secures by default is engineering time solving a solved problem

This project answers the question every cloud team needs to answer before any deployment:

> *"Who is responsible for this — Microsoft or us?"*

---

## Real Infrastructure Used

Rather than generic textbook examples, this analysis maps the model to actual infrastructure:

| Resource | Deployment Model | Description |
|---|---|---|
| **sykes-ubuntu** (ASUS X550ZA) | On-Premises | Physical laptop — analyst owns and manages everything |
| **sykeslogstorage** | PaaS | Azure Blob Storage — Microsoft manages infrastructure, analyst manages data and access |
| **Azure VM** *(Terraform project)* | IaaS | Planned — Microsoft manages physical layer, analyst manages OS and above |

---

## The Responsibility Matrix

Color key: 🟢 Microsoft · 🔵 You · 🟡 Shared

| Responsibility Layer | On-Prem (sykes-ubuntu) | IaaS (Azure VM) | PaaS (sykeslogstorage) |
|---|---|---|---|
| Physical hardware | 🔵 You | 🟢 Microsoft | 🟢 Microsoft |
| Physical network infrastructure | 🔵 You | 🟢 Microsoft | 🟢 Microsoft |
| Datacenter physical security | 🔵 You | 🟢 Microsoft | 🟢 Microsoft |
| Hypervisor | 🔵 You | 🟢 Microsoft | 🟢 Microsoft |
| Operating system | 🔵 You | 🔵 You | 🟢 Microsoft |
| Network controls (firewall/NSG) | 🔵 You | 🔵 You | 🟡 Shared |
| Applications | 🔵 You | 🔵 You | 🔵 You |
| Data | 🔵 You | 🔵 You | 🔵 You |
| Identity & access management | 🔵 You | 🔵 You | 🔵 You |
| Encryption configuration | 🔵 You | 🟡 Shared | 🟡 Shared |

### The Key Principle

> *"The higher up the service model stack you go from on-premises to IaaS to PaaS, the more responsibility shifts to Microsoft and the less you manage. But you always remain responsible for your **data** and your **identities** regardless of the deployment model."*

---

## Architecture — The Responsibility Stack

```
                    ON-PREM          IaaS             PaaS
                  sykes-ubuntu     Azure VM      sykeslogstorage
                  ┌──────────┐   ┌──────────┐   ┌──────────┐
Data              │   YOU    │   │   YOU    │   │   YOU    │
Identity/Access   │   YOU    │   │   YOU    │   │   YOU    │
Applications      │   YOU    │   │   YOU    │   │   YOU    │
Encryption        │   YOU    │   │  SHARED  │   │  SHARED  │
Network controls  │   YOU    │   │   YOU    │   │  SHARED  │
Operating system  │   YOU    │   │   YOU    │   │ Microsoft│
─────────────────────────────────────────────────────────────
Hypervisor        │   YOU    │   │Microsoft │   │Microsoft │
Datacenter sec.   │   YOU    │   │Microsoft │   │Microsoft │
Physical network  │   YOU    │   │Microsoft │   │Microsoft │
Physical hardware │   YOU    │   │Microsoft │   │Microsoft │
                  └──────────┘   └──────────┘   └──────────┘
```

The line separating **You** from **Microsoft** moves higher up the stack as you move right. This is the visual proof of why PaaS reduces operational overhead compared to IaaS, and why IaaS reduces it compared to on-premises.

---

## Key Decisions and Why

### 1. Using real infrastructure instead of textbook examples
Most shared responsibility analyses use fictional organizations. Using actual infrastructure forces genuine understanding — you can't pattern-match your way through a matrix built on infrastructure you actually operate. It also surfaces real-world nuance that generic examples miss.

### 2. Including on-premises as a column
Most cloud analyses skip on-premises entirely. Including it makes the cloud value proposition immediately visible. You can see exactly which responsibilities disappear when you lift a workload from a physical machine to Azure. This is especially relevant for federal IT environments where hybrid deployments are the norm during cloud migration.

### 3. The three cells that required the most reasoning

**Encryption — Shared in IaaS and PaaS:**
Microsoft provides encryption at rest by default. You control configuration choices (key management, TLS version, application-level encryption). Both contribute = Shared.
> *"Microsoft encrypts the floor, you lock the door."*

**Network controls — Shared in PaaS:**
Microsoft manages the underlying network infrastructure and provides platform-level DDoS protection. You configure access rules, private endpoints, and firewall policies. Both contribute = Shared.
> *"Microsoft owns the pipes, you control the valves."*

**Datacenter security — Microsoft in both IaaS and PaaS:**
Once any workload moves to the cloud, physical security is entirely Microsoft's. You cannot visit, audit, or influence Azure's physical controls.
> *"If you can't touch it, it's not yours."*

### 4. Real security gap identified during this analysis
Applying the shared responsibility model systematically to sykeslogstorage revealed the storage account had TLS 1.0 set as the minimum version — a genuine security misconfiguration. Since encryption configuration is the **analyst's responsibility** in PaaS, this was identified and remediated immediately:

```bash
az storage account update \
  --name sykeslogstorage \
  --resource-group LinuxLogsRG \
  --min-tls-version TLS1_2
```

This is the practical value of the model — it's a structured security audit tool, not just an exam topic.

---

## What I Would Do Differently at Production Scale

### Operationalize via Azure Policy
At home lab scale this lives in a document. At production scale (especially FedRAMP/CMMC environments), Azure Policy would automatically enforce the controls that fall under the analyst's responsibility — TLS minimums, allowed locations, required encryption settings — with continuous compliance assessment via Defender for Cloud.

### Customer-Managed Keys
The current setup uses Microsoft-managed encryption keys — appropriate for non-sensitive log data. For federal environments handling CUI or classified data, customer-managed keys (CMK) in Azure Key Vault would be required, shifting key custody from Microsoft to the organization.

### Private Endpoints for All PaaS Resources
sykeslogstorage currently accepts connections from the public internet (secured by RBAC and TLS). At production scale, every PaaS resource would use Private Endpoints, making resources reachable only from within a VNet and invisible to the public internet. This moves network controls from Shared toward You.

### Zero Trust Identity Architecture
At production scale in a federal environment, identity controls would include:
- Conditional Access with MFA required for all users
- Privileged Identity Management (PIM) just-in-time admin elevation
- Azure AD Identity Protection — risk-based conditional access
- No standing admin access, no shared credentials

### Formal RACI Documentation
The matrix here assigns responsibility to "Microsoft" or "You." At production scale, a formal RACI matrix would name specific teams, roles, and individuals, including the Authorizing Official (AO), ISSO, System Owner, and Cloud Service Provider as required for FedRAMP authorization.

---

## AZ-900 Alignment

This project directly covers the **Cloud Concepts** domain (25-30% of AZ-900):

| AZ-900 Objective | Coverage |
|---|---|
| Describe the shared responsibility model | Core subject of this entire project |
| Identify which cloud model is appropriate for each scenario | On-premises vs IaaS vs PaaS comparison |
| Describe consumption-based model | Context for why PaaS reduces operational overhead |
| Describe cloud service types (IaaS, PaaS, SaaS) | Three-column matrix structure |

---

## Related Projects

- [☁️ Linux to Azure — Phase 1: Automated Log Backup](#)
- [☁️ Linux to Azure — Phase 2: Portal Verification & Security Hardening](#)
- [🐧 Linux Lab Machine — Swapping Kali for Ubuntu](#)
- [🪟 Active Directory Home Lab — Phase 1](#)

---

## Documentation

Full project documentation (PDF) is available in this repository covering the business problem, color-coded responsibility matrix, key decision rationale, three tricky cells explained, and production-scale considerations.

---

*Bryan Sykes | Home Lab Portfolio | September 2026*
*[![LinkedIn](https://img.shields.io/badge/LinkedIn-SecuredByBryan-0A66C2?style=flat&logo=linkedin)](https://linkedin.com) [![GitHub](https://img.shields.io/badge/GitHub-MrBSykes-181717?style=flat&logo=github)](https://github.com/MrBSykes) [![X](https://img.shields.io/badge/X-@SecuredByBryan-000000?style=flat&logo=x)](https://x.com/SecuredByBryan)*
