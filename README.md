### GCP-BigQueryFinOpsDashboard

![big0](https://github.com/user-attachments/assets/ec951201-061a-428f-b546-c67b111a8f7e)

## Introduction

This project aims to create a FinOps dashboard using GCP's Recommender data to optimize costs. The recommender data will first be exported to BigQuery and automatically updated at intervals set by the user. After the export, we will create a filtered table containing only **"cost"** type recommendations by scheduling a query in BigQuery. This filtered table will be connected to a **Looker Studio** dashboard to visualize the data. Although it's possible to directly connect the Recommender table to Looker, a scheduled query is used to clean the data and focus solely on cost-related recommendations.

### 1. Recommendations export

- **Recommendations Hub > Data Transfer to BigQuery from Recommendations Hub**  
  - Configure Transfer  
    -- Transfer config name  
    -- Schedule options (Repeat frequency)  
    -- Destination settings (Dataset)
    
<img src="https://github.com/user-attachments/assets/50bf9539-9e19-4c17-a488-8743c0687bdf" alt="big1" width="600"/>

- **BigQuery > Create Transfer with Data Transfers**  
  - Source type: Recommender V1  
    - Configure Transfer  
      -- Transfer config name  
      -- Schedule options (Repeat frequency)  
      -- Destination settings (Dataset)  
      -- Data Source (Organization's recommendations)
      
  <img src="https://github.com/user-attachments/assets/d0174734-642b-463c-8f6d-6fdc13c4fac5" alt="big2" width="600"/>
  
### 2. BigQuery Environment: Scheduled Query to Filter Recommendations

Once the recommendations data is exported to BigQuery, you can access the resulting tables. The goal now is to create a scheduled query that will filter only the recommendations related to cost optimization. 

- **Steps to configure the scheduled query:**
  - Go to BigQuery and navigate to your dataset.
  - Click on **Scheduled Queries**.
  - Create a new scheduled query and paste the provided SQL.
  - Configure the schedule to run at intervals that fit your needs (e.g., daily, weekly).
  - Select the destination table where the cleaned recommendations will be stored.

Below is an example of a query that you can use to create a clean table with cost recommendations.

```sql
WITH filtered_recommendations AS (
   SELECT
       AS VALUE ARRAY_AGG(t ORDER BY last_refresh_time DESC LIMIT 1)[OFFSET(0)]
   FROM
       `your_project.your_dataset.recommendations_export` t
   WHERE
       t.primary_impact.category = 'COST'
       AND _PARTITIONTIME = (
           SELECT MAX(_PARTITIONTIME)
           FROM `your_project.your_dataset.recommendations_export`
       )
   GROUP BY
       t.name
)
SELECT
   t.*,
   p.project_id,  
   p.project_name, 
   REGEXP_EXTRACT(t.name, r'projects/([^/]+)') AS project_id,
   REGEXP_EXTRACT(t.name, r'locations/([^/]+)') AS location,
   REGEXP_EXTRACT(t.name, r'recommenders/([^/]+)') AS recommender_type,
   REGEXP_EXTRACT(t.name, r'recommendations/([^/]+)') AS recommendation_id,
   REGEXP_EXTRACT(ARRAY_TO_STRING(t.target_resources, ''), r'[^/]+$') AS target_resource_name,
   CAST(FORMAT('%.2f', 
    (t.primary_impact.cost_projection.cost_in_local_currency.units + 
    (t.primary_impact.cost_projection.cost_in_local_currency.nanos / 1000000000.0)) * -1) AS FLOAT64) AS potential_savings_per_month,
   FORMAT_TIMESTAMP('%d-%m-%Y', t.last_refresh_time) AS last_update
FROM
   filtered_recommendations t
LEFT JOIN
   `your_project.your_dataset.project_id_name_mapping` p
ON
   p.project_number = CAST(REGEXP_EXTRACT(t.name, r'projects/([^/]+)') AS INT64);
```

### 3. Looker Studio Dashboard

In this final step, we will connect the cleaned recommendations table from BigQuery to a Looker Studio dashboard. Although the raw recommendations could be connected directly, we use a filtered table to focus specifically on cost-related optimizations. 
