Block Diagram

```
  +----------------+        +-------------------+        +--------------------+        +------------------+
  |  Data Sources  |  -->   |  Azure Data Factory|  -->   |    Azure Synapse /   |  -->   |   Power BI / BI   |
  | (ADLS, SQL DB, |        |  (Ingestion +      |        |   Azure Databricks   |        |   Dashboards /    |
  |  Event Hubs,   |        |  Validation)       |        |  (Curated, Analytics)|        |   Reporting      |
  |  APIs, etc.)   |        +-------------------+        +--------------------+        +------------------+
  +----------------+               |                              |                                |
                                   |                              |                                |
                                   v                              v                                v
                         +-------------------+          +--------------------+           +-------------------+
                         | Raw Data Layer    |          | Curated Data Layer  |           | Gold/Analytics    |
                         | (ADLS Gen2 Blob)  |          | (Synapse Tables /   |           | Layer (Aggregates,|
                         +-------------------+          | Databricks Tables)  |           | Dashboards)       |
                                   |                     +--------------------+           +-------------------+
                                   |
                                   v
                      +-------------------------+
                      | Layer 1 Validation       |
                      | - Schema check           |
                      | - Basic structural check |
                      | (Optional Purview scan)  |
                      +-------------------------+
                                   |
                                   v
                      +-------------------------+
                      | Layer 2 Validation       |
                      | - ADF Data Flow          |
                      | - Runtime validations    |
                      | - Data cleaning & flags  |
                      +-------------------------+
                                   |
                                   v
                      +-------------------------+
                      | Layer 3 Validation       |
                      | - Azure Function         |
                      | - Complex JSON Schema    |
                      |   validations            |
                      +-------------------------+
                                   |
                                   v
                      +-------------------------+
                      | Data Quality Checks      |
                      | (PySpark + Deequ /       |
                      |  Great Expectations)     |
                      +-------------------------+
                                   |
                                   v
                            +-------------+
                            | Clean Data  |
                            +-------------+
  
                        -----------------------------------------
                                    Monitoring & Alerting
                        -----------------------------------------
                                   |          |
                                   v          v
                +----------------+              +----------------+
                | Azure Monitor  |              | Azure Log      |
                | Alerts         |              | Analytics      |
                +----------------+              +----------------+
                        |                                |
                        v                                v
           +---------------------------+    +---------------------------+
           | Email, SMS, Webhook, etc. |    | Centralized Log Queries &  |
           | Notifications             |    | Visualizations             |
           +---------------------------+    +---------------------------+
  
                        -----------------------------------------
                                   Microsoft Purview
                        -----------------------------------------
                    (Data Catalog, Lineage, Classification,
                     Business Glossary, Data Governance)


```


- Deequ vs Layer 2 & Layer 3

| Data Size           | Best Tool             | Why                           |
| ------------------- | --------------------- | ----------------------------- |
| < 1M rows           | ADF / Azure Function  | Low latency, simpler jobs     |
| 1M–50M rows         | ADF (simple) or Deequ | Depends on complexity         |
| 50M+ rows or >10 GB | ✅ Deequ (Databricks)  | Parallelism, metric profiling |

- Deequ requires 20-30s time to startup

