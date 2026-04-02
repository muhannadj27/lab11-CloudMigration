# Week 11 – Application Migration Lab
## Tailwind Traders: 3-Tier On-Premises to Azure Migration Plan

---

## Task 1 – Set Up Tooling for Discovery

### Discovery Method: Agentless + Agent-Based Mix

Tailwind Traders should use a **mixed approach**:

- **Agentless discovery** for WEB01, WEB02, and APP01 — these are VMware-hosted VMs, which Azure Migrate supports natively through agentless discovery via the vCenter API. No software needs to be installed on the VMs.
- **Agent-based discovery** for SQL01 — since SQL01 runs on a physical server (not VMware), the Azure Migrate appliance cannot discover it agentlessly. The Microsoft Monitoring Agent (MMA) and Dependency Agent must be installed directly on the physical machine.

### Number of Appliances Required

**1 Azure Migrate appliance** is sufficient for this environment. A single appliance can discover up to 10,000 VMware VMs and connect to one vCenter Server instance. Since Tailwind has only 4 servers (WEB01, WEB02, APP01, SQL01), one appliance handles the entire environment.

The appliance is deployed as an OVA template inside the VMware environment and communicates with Azure Migrate in the cloud.

### Credentials Required

| Target | Credential Type | Purpose |
|---|---|---|
| VMware vCenter | Read-only vCenter account | Agentless VM discovery for WEB01, WEB02, APP01 |
| Windows servers (WEB01, WEB02, APP01) | Local or domain admin account | Software inventory (installed apps, roles, features) |
| SQL01 (physical) | Windows admin + SQL sysadmin account | SQL Server discovery, version, databases, login modes |
| All servers | Domain credentials (if domain-joined) | Dependency mapping via MMA/Dependency Agent |

### 3 Best Practices During Discovery

1. **Run discovery for a minimum of 24–48 hours before assessment.** This ensures the appliance captures peak and off-peak usage patterns, giving accurate CPU, memory, and storage utilization data for right-sizing recommendations.

2. **Do not modify or reboot servers during the discovery period.** Any changes to the environment mid-discovery can skew performance data and produce inaccurate assessment results.

3. **Validate that all credentials have the correct permissions before starting.** Missing permissions on SQL or Windows accounts are the most common reason discovery fails silently — the appliance connects but returns incomplete data.

---

## Task 2 – Perform Assessment Planning

### Assessment Type: Production

Tailwind Traders should select a **production assessment**. The application is business-critical (it has a strict 1-hour downtime window and nightly backup requirements), which means the assessment must reflect real production load, not a test environment. Using non-production settings would result in undersized Azure VMs that cannot handle actual traffic.

### Target Region

**Canada Central** — Tailwind Traders is a Canadian company (based on the course context). Canada Central provides data residency within Canada, which is important for compliance, and has full support for the Azure services needed (VMs, Azure SQL, Load Balancer).

### Performance History Duration

**30 days** — this captures at least 4 full weekly cycles including weekday/weekend traffic differences and any monthly batch processes. A shorter window (e.g., 1 day) risks missing peak load periods.

### Sizing: Performance-Based vs On-Premises

**Performance-based sizing** should be used. Here is why:

On-premises servers are frequently over-provisioned — companies buy hardware for peak capacity years in advance and actual utilization is often 20–40% of total capacity. Using on-premises VM size would simply replicate that waste in Azure and result in unnecessarily expensive VMs.

Performance-based sizing uses the actual CPU, memory, and disk utilization data collected during discovery to recommend the smallest Azure VM that can comfortably handle the real workload. This typically reduces costs by 30–50%.

### Comfort Factor, Pricing Model, and Licensing

| Setting | Choice | Reason |
|---|---|---|
| Comfort factor | 1.3 (30%) | Adds a 30% buffer above peak observed utilization to handle traffic spikes without over-provisioning |
| Pricing model | Pay-as-you-go initially, then Reserved Instances after 3 months | Validates sizing in production before committing to 1- or 3-year reservations |
| Licensing model | Azure Hybrid Benefit | Tailwind already owns Windows Server and SQL Server licenses — Hybrid Benefit reuses them on Azure, saving up to 40% on VM costs |

---

## Task 3 – Dependency Analysis

### Application Components

| Component | Role | Type |
|---|---|---|
| WEB01 | Frontend web server | VMware VM |
| WEB02 | Frontend web server | VMware VM |
| LB01 | Internal load balancer (distributes traffic to WEB01/WEB02) | Hardware appliance |
| APP01 | API / application server | VMware VM |
| SQL01 | SQL Server 2017 database | Physical server |

### Identified Dependencies (minimum 8)

| # | Source | Destination | Port / Protocol | Service |
|---|---|---|---|---|
| 1 | LB01 | WEB01, WEB02 | TCP 80 / 443 | HTTP/HTTPS — inbound user traffic distributed to frontend |
| 2 | WEB01 | APP01 | TCP 8080 | REST API calls from frontend to application tier |
| 3 | WEB02 | APP01 | TCP 8080 | REST API calls from frontend to application tier |
| 4 | APP01 | SQL01 | TCP 1433 | SQL Server — application reads/writes to database |
| 5 | SQL01 | Backup server | TCP 445 | SMB — nightly backup job writes to network share |
| 6 | All servers | Active Directory DC | TCP/UDP 389 | LDAP — domain authentication |
| 7 | All servers | DNS server | UDP 53 | DNS resolution for internal hostnames |
| 8 | WEB01, WEB02 | External CDN / Internet | TCP 443 | HTTPS — serving static assets or calling external APIs |
| 9 | APP01 | SMTP gateway | TCP 25 | Email notifications sent by the application |
| 10 | All servers | WSUS / Patch server | TCP 8530 | Windows Update services |

### Items to Filter Out as Noise

During dependency mapping, the following connections should be **excluded** from migration planning as they are infrastructure noise rather than application dependencies:

- **Antivirus / endpoint agent traffic** — frequent polling to security management servers is not an application dependency
- **Monitoring agent heartbeats** — SCOM, Datadog, or similar tools generate constant low-level traffic
- **Windows time sync (NTP, UDP 123)** — standard OS behaviour, not application-specific
- **ICMP ping traffic** — network monitoring pings between all servers should be ignored

### Business Requirements from Application Owner

| Requirement | Detail |
|---|---|
| Business criticality | High — the application is customer-facing and revenue-generating |
| Uptime requirement | 99.9% SLA required (approximately 8.7 hours downtime per year) |
| Downtime window | Maximum 1 hour for planned maintenance or migration cutover |
| Data classification | Sensitive — customer PII and transaction data stored in SQL01 |
| Licensing dependencies | SQL Server 2017 Enterprise license (must be accounted for in Azure via Hybrid Benefit) |
| Patching requirements | Patches applied monthly during the approved maintenance window; no ad-hoc reboots |
| Firewall & IP considerations | Firewall rules currently allow only specific IP ranges; these must be replicated as Azure NSG rules. Internal DNS names must be preserved or updated in connection strings. |

---

## Task 4 – Validate Assessment Results with Application Owner

### Recommended Azure VM Sizes

| Server | Current Role | Recommended Azure VM | Reason |
|---|---|---|---|
| WEB01, WEB02 | Frontend web servers | **Standard_D2s_v3** (2 vCPU, 8 GB RAM) | Web servers are typically CPU-light and stateless; D-series general purpose is sufficient |
| APP01 | API / application server | **Standard_D4s_v3** (4 vCPU, 16 GB RAM) | Application tier handles business logic and needs more memory for in-memory processing |
| SQL01 | SQL Server 2017 | **Standard_E8s_v3** (8 vCPU, 64 GB RAM) | SQL workloads are memory-intensive; E-series memory-optimized VMs are the correct family |

### Components Needing Replacement or Optimization

- **LB01 (hardware load balancer)** — must be replaced with **Azure Load Balancer** (for internal Layer 4 traffic) or **Azure Application Gateway** (if Layer 7 routing or WAF is needed). The physical hardware appliance cannot be migrated to Azure.
- **SQL01 (physical SQL Server)** — consider migrating to **Azure SQL Managed Instance** instead of rehosting as a VM, which would eliminate patching overhead and provide built-in HA. See Task 4 SQL options below.
- **Nightly backup solution** — the current SMB-based backup to a network share should be replaced with **Azure Backup** integrated with the Azure VM or SQL MI backup policy.
- **Firewall rules** — all existing firewall rules must be recreated as **Azure Network Security Groups (NSGs)** on the relevant subnets.

### Dependency Validation

All 10 dependencies identified in Task 3 have been reviewed:
- Internal dependencies (WEB → APP → SQL) will be maintained within the same Azure Virtual Network
- Active Directory integration will be handled via **Azure AD DS** or extending on-premises AD via VPN/ExpressRoute
- SMTP gateway dependency must be replaced with **Azure Communication Services** or an external SMTP relay
- External CDN connections remain unchanged as they are outbound internet traffic

### SLA and Downtime Validation

- Azure VMs with **Availability Sets** (WEB01 + WEB02 in the same set) provide 99.95% SLA — this exceeds the 99.9% requirement
- The 1-hour cutover window is achievable using Azure Site Recovery replication (pre-sync data before cutover, then flip DNS)
- SQL01 migration requires the most time — a test migration must be completed first to validate the cutover fits within 1 hour

### SQL Migration Options

| Option | Description | Best For |
|---|---|---|
| **Rehost (IaaS VM)** | Lift SQL Server 2017 as-is onto an Azure VM | Fastest migration, no application changes needed, full SQL Server feature compatibility |
| **Azure SQL Managed Instance** | Fully managed SQL Server-compatible PaaS service | Best long-term option — eliminates OS/SQL patching, built-in HA and backups, ~99% compatibility with SQL Server |
| **Azure SQL Database** | Fully managed, serverless/elastic PaaS database | Only suitable if the application uses a small subset of SQL features; requires application code changes |

**Recommendation for Tailwind:** Start with **Rehost** to meet the 1-hour downtime requirement, then plan a Phase 2 migration to **Azure SQL Managed Instance** after stabilization.

---

## Task 5 – Migration Plan (Runbook)

### Pre-Migration Tasks

- [ ] Deploy Azure Migrate appliance and complete discovery and assessment
- [ ] Create target Azure Virtual Network with subnets matching on-premises network layout
- [ ] Configure Azure NSGs replicating all existing firewall rules
- [ ] Set up Azure Bastion for secure VM access post-migration
- [ ] Configure Azure Site Recovery (ASR) replication for WEB01, WEB02, APP01
- [ ] Install Azure Migrate agent on SQL01 (physical server)
- [ ] Perform test migration of all VMs into an isolated test VNet — validate application functionality
- [ ] Notify stakeholders and confirm the 1-hour downtime window
- [ ] Freeze all code deployments and configuration changes 48 hours before cutover
- [ ] Take full backup of SQL01 database and verify restore

### Migration Steps by Server Group

**Wave 1 — WEB01, WEB02 (Frontend)**
1. Enable ASR replication for WEB01 and WEB02 — allow initial sync to complete (24–48 hrs before cutover)
2. At cutover start: pause application traffic at LB01
3. Trigger ASR failover for WEB01 and WEB02
4. Validate VMs are running in Azure and can reach APP01
5. Update Azure Load Balancer backend pool with new VM IPs

**Wave 2 — APP01 (Application)**
1. Enable ASR replication for APP01
2. At cutover: trigger ASR failover for APP01
3. Validate APP01 can reach SQL01 on TCP 1433
4. Update connection strings in APP01 config to point to Azure SQL or new SQL01 IP

**Wave 3 — SQL01 (Database)**
1. Use SQL Server backup/restore or Database Migration Service to migrate SQL databases to Azure VM
2. Validate all databases are intact and accessible
3. Run application smoke tests against the migrated database
4. Update APP01 connection strings to point to new SQL01 Azure VM

### DNS Updates
- Update internal DNS records for WEB01, WEB02, APP01, SQL01 to point to new Azure private IPs
- Update external DNS (if public-facing) to point to Azure Load Balancer public IP
- TTL should be lowered to 60 seconds 24 hours before cutover to speed up propagation

### Connection String Changes
- APP01 → SQL01: update connection string from physical server IP to new Azure VM private IP
- If moving to Azure SQL MI: update connection string to use the MI endpoint FQDN
- All connection strings should be stored in Azure Key Vault rather than config files

### Load Balancer Considerations
- LB01 (hardware) is replaced by **Azure Load Balancer** in the Standard tier
- Configure health probes on TCP 80/443 pointing to WEB01 and WEB02
- Session persistence (sticky sessions) must be configured if the application requires it
- Azure Application Gateway should be evaluated if WAF or SSL termination is needed

### SQL Migration Considerations
- Perform a full database backup before cutover
- Use **Azure Database Migration Service** for minimal-downtime migration if the 1-hour window is at risk
- Validate SQL Agent jobs, linked servers, and logins are migrated
- Re-apply SQL Server firewall rules as Azure NSG rules on the SQL subnet

### Post-Migration Validation Checklist
- [ ] All VMs running and reachable via Azure Bastion
- [ ] Load balancer health probes showing all backends healthy
- [ ] End-to-end application test: user login → browse → add to cart → checkout
- [ ] SQL01 databases accessible and returning correct data
- [ ] Nightly backup job configured and tested in Azure Backup
- [ ] NSG rules verified — no unintended open ports
- [ ] Monitoring and alerts configured in Azure Monitor
- [ ] Performance metrics reviewed after 24 hours under real traffic

### Back-Out Plan
If migration fails or critical issues are found within the 1-hour window:

1. Trigger ASR **failback** for WEB01, WEB02, APP01 — this reverses replication back to on-premises VMware
2. Restore SQL01 from the pre-migration backup taken before cutover
3. Revert DNS records to original on-premises IPs (pre-lowered TTL ensures fast propagation)
4. Notify stakeholders of rollback and schedule a post-mortem
5. On-premises environment must remain fully operational and untouched until 48 hours after successful Azure validation

---

## Task 6 – Migration Waves

### Wave Grouping Justification

Migration waves are ordered based on three principles: dependency order (dependent services migrate after what they depend on), business risk (lowest risk first), and rollback complexity (easiest to roll back first).

**WEB01 and WEB02** are stateless frontend servers — they hold no persistent data and can be rolled back instantly via ASR failback. They have no dependency on APP01 or SQL01 being in Azure first (they call APP01 by hostname which can be updated). They are the natural starting point.

**APP01** depends on SQL01 being accessible but can temporarily point to the on-premises SQL01 during its migration window. It migrates second once the web tier is validated.

**SQL01** is the highest-risk component — it holds all persistent data, has the longest validation requirement, and a failed SQL migration is the hardest to recover from. It migrates last after the application tier is fully validated.

### Final Wave Table

| Wave | Servers | Reason |
|---|---|---|
| Wave 1 | WEB01, WEB02 | Stateless frontend servers — no persistent data, easy rollback via ASR failback, lowest risk. Migrated together behind Azure Load Balancer. |
| Wave 2 | APP01 | Application tier depends on web tier being stable in Azure first. Can temporarily connect back to on-premises SQL01 during validation period. |
| Wave 3 | SQL01 | Highest risk — contains all persistent customer and transaction data. Migrated last after full application validation. Requires longest testing window and has the most complex back-out procedure. |
