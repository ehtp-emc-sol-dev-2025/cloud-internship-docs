# Azure Multi-Region Web Application Architecture

## 1. Introduction and Goals

This document describes the design and components of the multi-region web application architecture. This design employs a **hybrid Active-Active Web Tier** and an **Active-Passive Data Tier** to balance performance, high availability, and data consistency.

* **Primary Goal:** Achieve **High Availability (HA)**, **Disaster Recovery (DR)**, and **low-latency content delivery** for global users.
* **Strategy:** Utilize two separate Azure regions (Region 1: Primary Write Region; Region 2: Secondary/Failover Region).

## 2. Architecture Diagram

![System Design](images/Architecture%20Diagram.png)


## 3. Component Breakdown and Role

| Service Name | Azure Service | Status in Both Regions | Function / Rationale |
| :--- | :--- | :--- | :--- |
| **Global Load Balancer** | Azure Front Door | **Active (Latency Routing)** | Global entry point, Caching, WAF, and routes traffic to the fastest (closest) regional endpoint. Provides instant failover for the web tier. |
| **Frontend** | Azure Static Web Apps | **Active** | Hosts the client-side UI. Decoupled from the API for fast deployment and scaling. Serves users from the closest region. |
| **Backend API** | App Service API | **Active** | Hosts the core business logic. All instances connect to the single SQL Write Master via the Failover Listener. |
| **Relational DB** | Azure SQL Database | **R1: Primary (Read/Write)** $\newline$ **R2: Secondary (Read-Only)** | Mission-critical structured data. Uses a **Failover Group** to manage replication and enable fast failover. |
| **SQL Failover Listener**| Azure SQL DB Feature | Global Endpoint | Provides a stable, unchanging connection string that always points to the current **Primary** database, regardless of its location (R1 or R2). |
| **NoSQL DB** | Azure Cosmos DB | **Multi-Region (Configurable)** | Used for non-relational, high-throughput data (e.g., session state). Can be configured for local reads in both R1 and R2. |
| **Static Content** | Azure Blob Storage & CDN | **Global** | Storage for static files (images, videos). CDN ensures low-latency delivery worldwide by caching content closer to the user. |

## 4. Operational Flow (Hybrid Active/Active Web + Active/Passive Data)

### 4.1. Normal Operation

1.  **User Request:** The user connects to the **Azure Front Door** using the HTTPS custom domain.
2.  **Web Tier Routing (Active-Active):** Front Door uses **Latency-based routing** to forward the request to the geographically **closest** active web endpoint (either App Service API in **Region 1 or Region 2**).
3.  **Read Operations:** The receiving App Service API (R1 or R2) can perform **local read operations** against its regional database replica (SQL Secondary in R2, SQL Primary in R1) or Cosmos DB endpoint.
4.  **Write Operations:** Any transaction requiring a write or update (from R1 or R2):
    * The App Service API connects using the **SQL Failover Listener** endpoint.
    * The Listener directs the request to the current **SQL Primary** (which is in **Region 1** under normal operation).
    * Data is written to the Primary (R1) and then synchronously or asynchronously replicated to the Secondary (R2).

### 4.2. Disaster Recovery / Failover Process

1.  **Web Tier Failure (Region 1 App Service fails):**
    * Front Door detects the health probe failure in Region 1 and automatically stops sending new traffic to R1.
    * 100% of user traffic is instantly rerouted to the healthy **Region 2** endpoints (Static Web App R2 / App Service API R2).
2.  **SQL Database Failover (Region 1 SQL fails):**
    * The Azure SQL Database **Failover Group** is activated (manual or automatic).
    * The **SQL Secondary** in Region 2 is promoted to the new **Primary (Read/Write)**.
    * The **SQL Failover Listener** automatically updates its internal record to point all traffic to the new Primary in **Region 2**.
3.  **Business Continuation:** The application, now running entirely from **Region 2**, continues to process reads and writes using the same listener endpoint, ensuring minimal downtime and no required application connection string changes.

## 5. Security and Resiliency Considerations

* **Global Security:** **Azure Front Door Web Application Firewall (WAF)** is enabled to protect all endpoints against common web vulnerabilities (OWASP Top 10).
* **Data Resiliency:** The SQL Failover Group provides an established Recovery Point Objective (RPO) and Recovery Time Objective (RTO) for mission-critical data.
* **Networking:** [Describe VNet Integration or Private Endpoint use, e.g., All PaaS services are integrated into a Virtual Network using Private Link to prevent public internet access.]
* **Authentication:** [Describe the identity provider, e.g., User authentication is managed via Azure Active Directory (Entra ID) and JWTs for API calls.]

## 6. Next Steps and Improvement Areas

* **Monitoring and Alerting:** Implement **Azure Monitor** and **Application Insights** across both regions to track performance, latency, and proactive failure detection.
* **Cost Optimization:** Review capacity planning and evaluate the use of **Azure Reservations** for App Service and SQL Database to optimize long-term operational costs.
* **Testing:** Regularly perform **Disaster Recovery Drills** to test the full failover and failback process, ensuring the RTO and RPO targets are met.
