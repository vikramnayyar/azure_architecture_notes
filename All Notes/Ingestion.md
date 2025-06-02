
- Data Ingestion (with Schema Validations & Cleaning)
  - 1. Goal 
    - Goal 1 --load--> Raw Data ----> ADLS
    - Goal 2 --def--> Schema --downstream--> ETL & DQ

  - 2. Tools for Ingestion
       
      | Source Type                | Ingestion Tool                                                                      |
      | -------------------------- | ----------------------------------------------------------------------------------- |
      | Databases (SQL, NoSQL)     | **Azure Data Factory (ADF)** copy activity                                          |
      | APIs / Streaming           | **Azure Event Hub** or **Azure Stream Analytics**                                   |
      | Files (CSV, JSON, Parquet) | **ADF**, **AzCopy**, **ADF Mapping Data Flows**, or custom Spark jobs in Databricks |

 
  - 3. ADLS Gen2 
    - Hier NameSpaces --enables--> Partitions
    - Databases --support--> 1. SQL    2. NoSQL
    - stores --data--> 1. Raw   2. Cleaned   3. Curated
 
   
    - 3.1 Partitions Understanding
    
      | Partition Column       | When to Use It                             | Example             |
      | ---------------------- | ------------------------------------------ | ------------------- |
      | `week` or `date`       | Almost always                              | `.../2025/week_22/` |
      | `retailer`             | If analyzing by channel                    | `.../walmart/`      |
      | `brand` or `category`  | For brand-specific dashboards or analytics | `.../toothpaste/`   |
      | `region` or `store_id` | For geo-segmented reporting                | `.../usa/west/`     |


      | Do                                                  | Don’t                                                       |
      | --------------------------------------------------- | ----------------------------------------------------------- |
      | Use week/date-based partitions for time-series data | Don’t partition by high-cardinality fields like `SKU_ID`    |
      | Match partitioning to your most common filters      | Don’t use hourly partitions unless you truly need streaming |
      | Keep partitions manageable in size (100MB–1GB)      | Avoid creating thousands of tiny files                      |

      🧠 Rule of Thumb:
      If you're filtering by a field frequently in queries, or it's a common join key, it’s a good candidate for partitioning.
      Industry Practice ----> Partition --every--> 100mb-1gb parquet files

      🧠 CPG Insight:
      Most advanced CPG data teams (like at Unilever, PepsiCo, Nestlé) partition by week + region + retailer, because that’s how they measure KPIs like market share, distribution, ROI, and lift.
      
   - 3.2 Partition Size Sense : Business Question Example
     - Brand X --how_performed--> Last 4 Weeks (in South Region)
     - partition --should_be--> `/curated/pos/retailer=walmart/region=south/week=2025-W20/`
     - Folder Traversals ----> 4
       - if --parition_less--> Scan 500GB files
       - if --parition_large--> traverse ----> large folders (+meta data overhead)
      
   - 3.3 Zoning Files          
      | Zone        | Purpose                  | CPG Example                                             |
      | ----------- | ------------------------ | ------------------------------------------------------- |
      | **Raw**     | Preserve original format | Store IRI `.csv` deliveries as-is (for audit)           |
      | **Cleaned** | Clean, join, standardize | Deduplicate POS by UPC; join with master SKU list       |
      | **Curated** | Analytics-ready datasets | `market_share_by_retailer.parquet` for Tableau/Power BI |



  - 4. Data Sources in CPG Analytics
      - 4.1 
  
      | Data Type      | Source Examples                | Volume/Frequency | Use Case                           |
      | -------------- | ------------------------------ | ---------------- | ---------------------------------- |
      | **POS data**   | Nielsen, IRI, Retailer Portals | Weekly, GBs–TBs  | Market share, pricing              |
      | **ERP data**   | SAP, Oracle                    | Daily            | Demand planning, order fulfillment |
      | **E-commerce** | Amazon, Shopify APIs           | Real-time        | Omnichannel insights               |
      | **Marketing**  | Meta, Google Ads, CRMs         | Daily/hourly     | Attribution, ROI                   |
      | **IoT/Sensor** | Cold chain, vending machines   | Streaming        | Operational analytics              |

    - 4.2 Data Source vs Tool & When to Use Them
    
      | **Source Type**                     | **Tool**                                        | **CPG Example**                                  |
      | ----------------------------------- | ----------------------------------------------- | ------------------------------------------------ |
      | **SQL/NoSQL databases**             | **Azure Data Factory (ADF) - Copy Activity**    | Pull `sales_orders` from SAP HANA daily          |
      | **APIs / Streaming**                | **Azure Event Hub or Azure Stream Analytics**   | Real-time ad clickstream ingestion from Meta Ads |
      | **Flat files (CSV, Parquet, JSON)** | **ADF, AzCopy, Databricks, Mapping Data Flows** | Weekly Nielsen POS delivery (often \~5GB+)       |
  
      - CPG Firms ----> Weekly Syndicated Files -----> 3-10GB ----> 100k rows per SKU
     

  - 5. Layers in Schema Validation
    - Layers in Ingestions (3 Schema Validations Followed by Other DQ Validations) 
        ```
        [Layer 1] JSON or other Schema File – types, required, nullable
            ⮕ Optional pre-check (external or custom) for basic structure    
             ↓
    
        📦 Raw Layer (store as-is, maybe even some bad data)
              ----> Best Practice: Store after Step 1 - Audit Compliant (Optional - do this after 3rd layer) 
             ↓
    
        [Layer 2] ADF Data Flow – minLength, regex, value logic
            ⮕ Real-time validation inside transformations
            ⮕ Use expressions like: length(), regexMatch(), iif(), etc.
    
            ↓
    
        [Layer 3] ADF Pipeline (Azure Function) – full JSON Schema validation, "format": "email"
            ⮕ External service call for strict validation (e.g., format: email, uri)
            ⮕ Returns pass/fail JSON with error message
    
             ↓
    
        🔍 **PySpark + Deequ (or Great Expectations)** – rich data quality checks
            ⮕ Run in Azure Synapse, Azure Databricks, or HDInsight
            ⮕ Checks like:
               - Completeness
               - Uniqueness
               - Referential integrity
               - Value distributions
               - Data drift or outliers
            ⮕ Generate validation report & score
        
             ↓
    
        Clean Data
    
             ↓
    
        Curated Layer
        
        ```
        
        ```
        ✖ Invalid Records
            ⮕ Route to Quarantine/Rejected folder
            ⮕ Include error reason (e.g., "username too short", "invalid email")
        ``` 



     
- Microsoft Purview
  - Microsoft Purview ----> 1. Cataloging    2. Data Governance
  - Schema Discovery --scans--> 1. ADLS 2.SQL 3.Synapse --extracts_structure--> Tables | Columns | Data Types etc
  - Data Classification ----> Financial Info | PII (Personally Identifiable Info like emails, names)
    - Imp --imply_regulations--> GDPR or HIPAA etc --control_access--> sensitive data
  - Data Lineage Tracking --for--> Auditing | debugging
  - Glossary and Business Metadata --dict--> Business Terms ----> Brands | Category | SKU etc --for--> Convinience (of Non tech users)

    ```
    +----------------+       +----------------+       +----------------+       +----------------+
    |  Data Sources  |  -->  |   Azure Data   |  -->  |    Azure       |  -->  |    Power BI    |
    | (ADLS, SQL,    |       |   Factory (ADF)|       |   Synapse      |       |   Dashboards   |
    | Synapse, etc.) |       |  (Orchestration)|       | (Data Warehouse)|       | (Reporting)   |
    +----------------+       +----------------+       +----------------+       +----------------+
             |                        |                       |                        |
             |                        |                       |                        |
             |                        |                       |                        |
             v                        v                       v                        v
    +------------------------------------------------------------------------------------+
    |                            Microsoft Purview Platform                              |
    |                                                                                    |
    |  - **Schema Discovery:** Scans data sources and extracts schema (tables, columns)  |
    |  - **Data Classification:** Tags sensitive data (PII, financial info)              |
    |  - **Lineage Tracking:** Maps data journey across all stages                        |
    |  - **Glossary/Metadata:** Business terms (Brand, SKU, Region) linked to data       |
    +------------------------------------------------------------------------------------+
    ```
 
- Azure Alerting
  - ADF --reads--> Source --compare_schema--> Schema File --types--> Config File | 
    - Config File ----> json | yaml ----> simple, portable, outside Azure --used--> smaller projects --datasets_increase--> Unmanageable | NO Advanced Queries (like SQL on dbs) --also_lack--> Permissions
  - Azure SQL Meta Data --adv--> Centralized | Complex Queries | Permissions --complex_working--> Db Skills | Harder Changes --only-- Azure
  - JSON ----> INdustry Standard ----> 1. Detailed Schema  --for--> Heavy Data Sources --more_used--> Hiererchical Data (Non_tabular) 

  - JSON Schema Example
  ```
  import json
  import logging
  import azure.functions as func
  from jsonschema import validate, ValidationError, FormatChecker
  
  # Define your JSON schema with "format": "email"
  schema = {
      "type": "object",
      "properties": {
          "email": {
              "type": "string",
              "format": "email"
          },
          "age": {
              "type": "integer",
              "minimum": 18,
              "maximum": 99
          }
      },
      "required": ["email", "age"]
  }
  
  def main(req: func.HttpRequest) -> func.HttpResponse:
      logging.info('Azure Function: JSON Schema validation triggered.')
  
      try:
          data = req.get_json()
      except ValueError:
          return func.HttpResponse(
              json.dumps({"status": "invalid", "error": "Invalid JSON"}),
              status_code=400,
              mimetype="application/json"
          )
  
      try:
          # Validate using jsonschema with FormatChecker enabled
          validate(instance=data, schema=schema, format_checker=FormatChecker())
          return func.HttpResponse(
              json.dumps({"status": "valid"}),
              status_code=200,
              mimetype="application/json"
          )
      except ValidationError as e:
          return func.HttpResponse(
              json.dumps({"status": "invalid", "error": e.message}),
              status_code=400,
              mimetype="application/json"
          )
  ```
 
- All Inbuilt Formats in Azure (for Schema Validations)
  | Format          | Description                           | Example                            |
  | --------------- | ------------------------------------- | ---------------------------------- |
  | `date-time`     | RFC 3339 date-time format             | `"2023-06-01T13:45:30Z"`           |
  | `date`          | Full-date as per RFC 3339             | `"2023-06-01"`                     |
  | `time`          | Time of day, RFC 3339                 | `"13:45:30Z"`                      |
  | `email`         | Internet email address                | `"user@example.com"`               |
  | `hostname`      | DNS hostname                          | `"example.com"`                    |
  | `ipv4`          | IPv4 address                          | `"192.168.0.1"`                    |
  | `ipv6`          | IPv6 address                          | `"2001:0db8:85a3::8a2e:0370:7334"` |
  | `uri`           | Uniform Resource Identifier (URI)     | `"https://example.com"`            |
  | `uri-reference` | URI or relative URI                   | `"/path/resource"`                 |
  | `uri-template`  | URI template as per RFC 6570          | `"https://example.com/{id}"`       |
  | `json-pointer`  | JSON Pointer (RFC 6901)               | `"/foo/bar/0"`                     |
  | `regex`         | Regular expression pattern (ECMA 262) | `"^[A-Za-z0-9]+$"`                 |



  - Schema Checks Possible in Azure (crossed ones cannot be declared in Schema File)

    | Validation Type           | Description                                     | Example / Usage                               | Supported in ADF?                                            |
    |--------------------------|------------------------------------------------|-----------------------------------------------|--------------------------------------------------------------|
    | **Type Check**            | Check data type (string, integer, boolean, etc.) | Define schema with `"type": "string"`         | ✅ Supported via dataset schema                               |
    | **Required Fields**       | Ensure field presence                           | Specify `"required": ["fieldName"]`           | ✅ Supported in schema                                        |
    | **Minimum / Maximum**     | Numeric minimum and maximum value               | `"minimum": 0`, `"maximum": 100`              | ✅ Supported in Mapping Data Flow expressions                 |
    | **MinLength / MaxLength** | Minimum and maximum string length               | `"minLength": 5`, `"maxLength": 20`           | ❌ Not directly supported; can be implemented with expressions |
    | **Pattern (Regex)**       | Validate string with regex pattern              | `"pattern": "^[a-zA-Z0-9]+$"`                 | ✅ Supported via `regexMatch()` in Data Flows                 |
    | **Enum**                  | Allow only specific values                      | `"enum": ["active", "inactive"]`              | ❌ Not native, can implement via expression checks            |
    | **Unique Items**          | Ensure array items are unique                   | `"uniqueItems": true`                          | ❌ Not supported                                              |
    | **Format**                | Email, URI, date-time formats                   | `"format": "email"`                            | ❌ Not natively supported (use custom validation)             |
    | **Nullability**           | Allow or disallow null values                   | `"nullable": true`                             | ✅ Supported with schema settings                             |
    | **Length (Array)**        | Minimum or maximum number of items in array     | `"minItems": 1`, `"maxItems": 5`              | ❌ Not supported                                              |
    | **Custom Validation**     | Business logic checks                           | Use Data Flow expressions, conditional splits | ✅ Fully supported via expressions                            |

     


 
