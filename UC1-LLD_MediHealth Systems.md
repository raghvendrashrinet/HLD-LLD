# Low-Level Design (LLD / NDD)

**Document Reference:** LLD-AZ-DATA-2026-V1.0  
**Associated HLD:** HLD-AZ-DATA-2026-V1.0  
**Project:** Enterprise Healthcare Analytics Landing Zone (MediHealth Systems)  
**Classification:** Confidential / Internal Only  

---

## 1. Network Topology & Subnet Mapping

This section maps the complete CIDR block allocation and network layout across the Hub and Spoke VNets.
```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│ AZURE HUB VNET: vnet-hub-prod-01 (10.100.0.0/22)                                        │
│                                                                                         │
│  ┌─────────────────────────────────┐           ┌─────────────────────────────────────┐  │
│  │ GatewaySubnet (10.100.0.64/27)  │           │ AzureFirewallSubnet (10.100.0.0/26) │  │
│  │  [ExpressRoute Gateway]         │           │  [afw-prod-01] IP: 10.100.0.4       │  │
│  └────────────────┬────────────────┘           └──────────────────▲──────────────────┘  │
│                   │                                               │                     │
│  ┌────────────────┴────────────────┐           ┌──────────────────┴──────────────────┐  │
│  │ snet-dns-inbound                │           │ snet-dns-outbound                   │  │
│  │ (10.100.0.96/28)                │           │ (10.100.0.112/28)                   │  │
│  │  [DNS Inbound EP: 10.100.0.100] │           │  [DNS Outbound EP: 10.100.0.116]    │  │
│  └─────────────────────────────────┘           └─────────────────────────────────────┘  │
└──────────────────────────────────────────┬──────────────────────────────────────────────┘
                                           │ VNet Peering (Allow Forwarded Traffic)
┌──────────────────────────────────────────▼──────────────────────────────────────────────┐
│ AZURE DATA SPOKE VNET: vnet-data-prod-01 (10.100.4.0/22)                                │
│                                                                                         │
│  ┌───────────────────────────────────────────────────────────────────────────────────┐  │
│  │ Ingestion Subnet: snet-ingest-prod-01 (10.100.4.0/26)                             │  │
│  │  ├── [vm-shir-01] IP: 10.100.4.4                                                  │  │
│  │  └── [vm-shir-02] IP: 10.100.4.5                                                  │  │
│  └───────────────────────────────────────┬───────────────────────────────────────────┘  │
│                                          │                                              │
│  ┌───────────────────────────────────────▼───────────────────────────────────────────┐  │
│  │ Analytics & Storage Subnet: snet-data-prod-01 (10.100.4.64/26)                    │  │
│  │  ├── [pe-stmedihealth-blob] IP: 10.100.4.68                                       │  │
│  │  ├── [pe-stmedihealth-dfs]  IP: 10.100.4.69                                       │  │
│  │  ├── [pe-synapse-workspace] IP: 10.100.4.70                                       │  │
│  │  ├── [pe-synapse-sql]       IP: 10.100.4.71                                       │  │
│  │  └── [pe-adf-prod-01]       IP: 10.100.4.72                                       │  │
│  └───────────────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

### Subnet Allocation Table

| VNet Name         | Subnet Name          | Subnet CIDR   | Usable IPs | Assigned Service / Component |
|-------------------|----------------------|---------------|------------|------------------------------|
| vnet-hub-prod-01  | AzureFirewallSubnet  | 10.100.0.0/26 | 59         | Central Azure Firewall (10.100.0.4) |
| vnet-hub-prod-01  | GatewaySubnet        | 10.100.0.64/27| 27         | ExpressRoute Gateway |
| vnet-hub-prod-01  | snet-dns-inbound     | 10.100.0.96/28| 11         | Private DNS Resolver Inbound Endpoint (10.100.0.100) |
| vnet-hub-prod-01  | snet-dns-outbound    | 10.100.0.112/28| 11        | Private DNS Resolver Outbound Endpoint (10.100.0.116) |
| vnet-data-prod-01 | snet-ingest-prod-01  | 10.100.4.0/26 | 59         | SHIR Virtual Machines (vm-shir-01, vm-shir-02) |
| vnet-data-prod-01 | snet-data-prod-01    | 10.100.4.64/26| 59         | Private Endpoints (ADLS Gen2, Synapse, ADF) |

---

## 2. Private Endpoint & DNS Mapping

### DNS Resolution Flow
```
On-Prem Host          On-Prem DNS Server       Azure DNS Resolver           Azure Private DNS Zone
 (Client)              (10.200.0.53)            Inbound Endpoint              (privatelink.blob...)
    │                        │                    (10.100.0.100)                      │
    │── 1. Query DNS ───────►│                          │                               │
    │   stmedihealth.blob... │                          │                               │
    │                        │── 2. Conditional Forward►│                               │
    │                        │   (*.blob.core...)       │                               │
    │                        │                          │── 3. Lookup A-Record ────────►│
    │                        │                          │                               │
    │                        │                          │◄── 4. Return IP: 10.100.4.68 ─│
    │                        │◄── 5. Forward Response ──│                               │
    │◄── 6. Return IP ───────│                          │                               │
    │    (10.100.4.68)       │                          │                               │
```

### Endpoint Mapping Table

| Azure Resource     | Endpoint Name          | Private IP   | Target Private DNS Zone              | FQDN |
|--------------------|------------------------|--------------|--------------------------------------|------|
| ADLS Gen2 Blob     | pe-stmedihealth-blob   | 10.100.4.68  | privatelink.blob.core.windows.net    | stmedihealth.blob.core.windows.net |
| ADLS Gen2 DataLake | pe-stmedihealth-dfs    | 10.100.4.69  | privatelink.dfs.core.windows.net     | stmedihealth.dfs.core.windows.net |
| Synapse Analytics  | pe-synapse-workspace   | 10.100.4.70  | privatelink.dev.azuresynapse.net     | syn-medihealth.dev.azuresynapse.net |
| Synapse SQL Pool   | pe-synapse-sql         | 10.100.4.71  | privatelink.sql.azuresynapse.net     | syn-medihealth.sql.azuresynapse.net |
| Azure Data Factory | pe-adf-prod-01         | 10.100.4.72  | privatelink.datafactory.azure.net    | adf-medihealth.datafactory.azure.net |

---

## 3. User-Defined Route (UDR) Configuration

### Flow
```
[ On-Prem Database ] ◄── (Port 1433) ──┐
 (10.200.50.10)                        │
                                       │ 5. ExpressRoute Circuit
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│ HUB VNET                                                                                │
│                                                                                         │
│   [ ExpressRoute Gateway ]                                                              │
│            │                                                                            │
│            │ 4. Route via ExpressRoute                                                  │
│            ▼                                                                            │
│   [ Azure Firewall: afw-prod-01 ] (10.100.0.4)                                          │
│   ── L4 Network Rule: Allow 10.100.4.0/26 -> 10.200.50.10:1433                          │
│            ▲                                                                            │
└────────────┼────────────────────────────────────────────────────────────────────────────┘
             │ 3. VNet Peering Transit (Forwarded to Firewall)
┌────────────┼────────────────────────────────────────────────────────────────────────────┘
│ SPOKE VNET │                                                                            │
│            │                                                                            │
│   [ Route Table: rt-data-spoke-to-hub ]                                                 │
│   ── UDR Rule: Destination 10.200.0.0/16 -> Next Hop: 10.100.0.4 (Virtual Appliance)    │
│            ▲                                                                            │
│            │ 2. Match UDR Route for 10.200.50.10                                        │
│            │                                                                            │
│   [ SHIR VM: vm-shir-01 ] (10.100.4.4)                                                  │
│   ── 1. Originates Query Request to 10.200.50.10:1433                                   │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

### Route Table

| Route Table Name       | Destination Prefix | Next Hop Type       | Next Hop IP | Purpose |
|------------------------|--------------------|---------------------|-------------|---------|
| rt-data-spoke-to-hub   | 0.0.0.0/0          | Virtual Appliance   | 10.100.0.4  | Force outbound internet traffic to Firewall |
| rt-data-spoke-to-hub   | 10.200.0.0/16      | Virtual Appliance   | 10.100.0.4  | Force on-premises traffic to Firewall |
| rt-data-spoke-to-hub   | 10.100.4.0/22      | VNet Local          | N/A         | Local intra-spoke subnet routing |

---

## 4. Network Security Group (NSG) Rules Matrix

**NSG Name:** nsg-snet-ingest-prod-01 (Attached to snet-ingest-prod-01)

### Inbound Rules

| Priority | Rule Name          | Access | Protocol | Source Address   | Destination Address | Dest Port | Purpose |
|----------|--------------------|--------|----------|------------------|---------------------|-----------|---------|
| 100      | Allow_Azure_LB     | Allow  | Any      | AzureLoadBalancer| Any                 | Any       | Infra health probes |
| 200      | Allow_OnPrem_Admin | Allow  | TCP      | 10.200.10.0/24   | 10.100.4.0/26       | 3389      | Secure RDP access |
| 4096     | Deny_All_Inbound   | Deny   | Any      | Any              | Any                 | Any       | Catch-all block |

### Outbound Rules

| Priority | Rule Name          | Access | Protocol | Source Address   | Destination Address | Dest Port | Purpose |
|----------|--------------------|--------|----------|------------------|---------------------|-----------|---------|
| 100      | Allow_To_OnPrem_DB | Allow  | TCP      | 10.100.4.0/26    | 10.200.50.10/32     | 1433      | EHR ingestion |
| 110      | Allow_To_Storage_PE| Allow  | TCP      | 10.100.4.0/26    | 10.100.4.68/31      | 443       | Secure ingestion to Lakehouse |
| 120      | Allow_ADF_Relay    | Allow  | TCP      | 10.100.4.0/26    | ServiceTag:ServiceBus| 443      | SHIR heartbeat |
| 4096     | Deny_All_Outbound  | Deny   | Any      | Any              | Any                 | Any       | Catch-all block |

---

## 5. Azure Firewall Policy Rules

| Rule Collection | Source        | Destination         | Protocol / Ports | Action | Intent |
|-----------------|---------------|---------------------|------------------|--------|--------|
| Network Rule    | 10.100.4.0/26 | 10.200.50.10/32     | TCP / 1433       | Allow  | L4 packet filtering for DB queries |
| App Rule        | 10.100.4.0/26 | *.datafactory.azure.net | HTTPS / 443   | Allow  | L7 FQDN authorization for ADF |
| App Rule        | 10.100.4.0/26 | *.servicebus.windows.net | HTTPS / 443 | Allow  | L7 control channel for SHIR relay |

---

