# High-Level Design (HLD)

**Document Reference:** HLD-AZ-DATA-2026-V1.0  
**Project:** Enterprise Healthcare Analytics Landing Zone (MediHealth Systems)  
**Author:** Lead Cloud Solutions Architect  
**Classification:** Confidential / Internal Only  

---

## 1. Executive Summary & Business Objectives

MediHealth Systems requires a HIPAA-compliant cloud analytics platform in Azure to ingest daily Electronic Health Record (EHR) data from on-premises systems, transform it, and enable BI reporting for enterprise clinical analytics.

**Core Design Principles:**
- **Zero Public Access:** No cloud service will expose a public IP address or public internet endpoint.  
- **Least Privilege Routing:** Traffic must flow through centralized security controls.  
- **Hybrid Reachability:** Native, encrypted connectivity between on-premises databases and Azure ingestion compute.  

---

## 2. Architecture Diagram
```
[ ON-PREMISES DATA CENTER ]
EHR Database (10.200.0.0/16)
│
│ ExpressRoute / IPsec VPN
▼
┌─────────────────────────────────────────────────────────────┐
│                      AZURE REGION (EAST US)                 │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ HUB VNET (vnet-hub-prod-01: 10.100.0.0/22)            │  │
│  │                                                       │  │
│  │  ┌───────────────────┐     ┌───────────────────────┐  │  │
│  │  │ Azure Firewall    │ ◄─► │ Private DNS Resolver  │  │  │
│  │  │ (afw-prod-01)     │     │ (dns-res-prod-01)     │  │  │
│  │  └───────────────────┘     └───────────────────────┘  │  │
│  └──────────────────────────┬────────────────────────────┘  │
│                             │                               │
│                             │ VNet Peering (Non-Transitive) │
│                             ▼                               │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ DATA SPOKE VNET (vnet-data-prod-01: 10.100.4.0/22)    │  │
│  │                                                       │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │ Ingestion Subnet (snet-ingest-prod-01)          │  │  │
│  │  │  ── Self-Hosted Integration Runtime (SHIR VMs)  │  │  │
│  │  └───────────────────────┬─────────────────────────┘  │  │
│  │                          │ Private Endpoint           │  │
│  │  ┌───────────────────────▼─────────────────────────┐  │  │
│  │  │ Analytics & Storage Subnet (snet-data-prod-01)  │  │  │
│  │  │  ── ADLS Gen2 (Lakehouse: Bronze/Silver/Gold)   │  │  │
│  │  │  ── Azure Synapse Workspace                     │  │  │
│  │  │  ── Azure Data Factory (ADF)                    │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```


---

## 3. Network Topology & Boundaries

- **Hub-and-Spoke Pattern:** A central Hub VNet controls core security services, while a Spoke Data VNet houses isolated compute and storage components.  
- **Perimeter Security:** All outbound internet traffic from the Spoke VNet is routed through the Hub Azure Firewall for payload inspection and threat filtering.  
- **On-Premises Connectivity:** Traffic from the on-premises network enters the Hub VNet via an ExpressRoute Gateway, keeping enterprise networks off the public web.  

---

## 4. Core Component Functions

| Service | Tier / SKU | Purpose |
|---------|------------|---------|
| **Azure Firewall** | Premium | Inspects spoke egress traffic, enforces FQDN/URL filtering, and handles IDPS. |
| **Azure Private DNS Resolver** | Standard | Handles cross-premises DNS resolution between Azure Private DNS Zones and On-Premises Active Directory DNS. |
| **ADF Self-Hosted IR** | 2x Standard D4s_v5 VMs | Deployed inside the Ingestion Subnet to reach on-premises EHR databases over internal IP address space. |
| **Azure Data Lake Gen2** | StorageV2 (Block Blob) | Stores raw, cleansed, and curated datasets across Bronze, Silver, and Gold containers. Access restricted to Private Endpoints. |
| **Azure Synapse Analytics** | Enterprise Workspace | Runs data transformation jobs and executes SQL/Spark queries against Lakehouse data without public IP exposure. |

---

## 5. Security & Governance Baseline

- **Network Isolation:** Public Network Access property set to *Disabled* across ADF, Synapse, and ADLS Gen2 instances.  
- **Data in Transit:** Enforced TLS 1.2 minimum across all internal and private endpoints.  
- **Identity Control:** Azure Active Directory (Entra ID) with Role-Based Access Control (RBAC) and Managed Identities for service-to-service communication — eliminating stored connection credentials.  

---
