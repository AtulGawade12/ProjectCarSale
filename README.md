
# Car Sales Data Engineering Pipeline

#Project Overview
This project creates a data engineering pipeline using Medallion Architecture to extract car sales performance data from GitHub, transform and load it into Azure Data Lake Storage (ADLS) Gen2 and Azure SQL Database. The pipeline enables incremental data loading, data transformation, and creation of a dimensional data model in a star schema.

#Architecture
The pipeline follows the Medallion Architecture, consisting of the following layers:

- Bronze Layer: Raw data extracted from GitHub and stored in ADLS Gen2 using Azure Data Factory (ADF) in Parquet file format.
- Silver Layer: Transformed data stored in ADLS Gen2, accessible by data scientists for separate analysis using Databricks.
- Gold Layer: Dimensional data model in a star schema, stored in Delta format, serving as the single source of truth for downstream applications like Power BI.

#Tools and Technologies
- Azure Data Factory (ADF): Used for data extraction, transformation, and loading.
- Azure Data Lake Storage (ADLS) Gen2: Used for storing raw and transformed data.
- Databricks: Used for data transformation and creation of the Silver Layer.
- Azure SQL Database: Used for storing data.
- Power BI: Used for creating dashboards and visualizations.
- GitHub: Used for storing raw data.
- Parquet: Used for storing raw data in the Bronze Layer.
- Delta: Used for storing data in the Gold Layer.

#Pipeline Workflow
1. Extract raw data from GitHub using ADF.
2. Store raw data in the Bronze Layer of ADLS Gen2 using ADF.
3. Transform data using Databricks and store in the Silver Layer.
4. Create a dimensional data model in a star schema and store in the Gold Layer in Delta format.
5. Load data into Azure SQL Database.
6. Use Power BI to create dashboards and visualizations.

#Incremental Loading
The pipeline is designed to store incremental data, where:

- At the first run, all data is stored.
- At subsequent runs, only incremental data is stored.

#Access and Usage
Data scientists can access the Silver Layer for separate analysis. Downstream applications like Power BI can access the Gold Layer for creating dashboards and visualizations.

#Detailed Steps:

##1] Resource Group:
   Login to Azure portal and create one resource group which will store all the project related resources.

##2] Data Lake:
   - Now, create one storage account where we will store our all types of data (i.e. raw, transformed and serving layer)
   - Search, "Storage Account" in azure portal and create one storage account. Fill the necessary configuration. I have selected Redundancy as LRS (Locally Redundant 
     Storage), since it is the cheapest option as it will replicate the data within same data centre. Other options are GRS (Geo Redundant Storage), ZRS (Zone 
     Reduandant Storage) and GZRS (Geo Zone Redunadant Storage)
   - In the next configuration I have enabled hierarchical namespace in order to use ADLS (Azure Data Lake Storage) instead of Blob storage. The primary difference 
     between both is we can create containers and can create multiple folders within those containers in ADLS.

     ![image](https://github.com/user-attachments/assets/be647d80-b245-48a4-90e5-87cc53698268)

   - Create three containers namely, bronze, silver and gold which defines different layers in Medallion architecture.

##3] ADF:
   Create resource, ADF (Azure Data Factory).

##4] Azure SQL:
   - create resource, Azure SQL.
   - Select SQL databases as the deployment option. This will help Azure to manage all admin level tasks. ![image](https://github.com/user-attachments/assets/c2778a81-a218-49ea-a920-df558e22ebeb)
   - SQL managed instances: will give admin level access.
   - SQL virtual machines: will provide virtual machine with Azure SQL installed in it and will have all admin level roles and responsibility.
   - SQL databses -> Enter database name -> craete new server -> Authentication Method: Use both SQL & MS Entra authentication -> set server admin login ID and 
     password -> set admin -> workload environment: development -> networking -> connectivity method -> Public Endpoints -> Review + Create
   - Now, go to resourse group and open the newly created SQL server -> networking -> Public Access -> Allow Azure services and resources to access this server.
   - At the bottom of the page, you will see databse that you have created. Open the database and click on query editor from left menu. Enter admin login ID and 
     password, you will see Azure SQL query editor.

##5] Data Extraction:
   - Since we will be extracting data from GitHub and load it onto SQL database, we need to create one destination table (source_car_sale).
   - Go to ADF -> Author -> pipeline -> New pipeline (SourcePrep): We will extract the data stored in Github and will store it in source_car_sale table INCREMENTALLY 
     using this pipeline.
   - Go to managed tab -> linked services -> New -> HTTP -> provide GitHub base URL -> Authentication Type: Anonymous -> Test connection -> create. {This linked 
     service (ls_git)will connect ADF to GitHub}
   - Create another linked service (ls_sqlDB)to connect ADF to Azure SQL: New -> Azure SQL database -> server name -> database name -> user name -> password -> Test 
     connection -> create.
   - Go to SourcePrep pipeline -> create copy activity -> source (parameterised dataset) -> New -> HTTP -> csv -> ls_git -> Advanced -> open this dataset -> parameter        -> Name (load flag) -> connection -> Relative URL -> Dynamic Content -> paste the relative URL without file name (as file name will be dynamic I>E> changing 
     during incremental load) -> @{select parameter (load flag)} -> give the value for load flag parameter source file name...... this is how you will create 
     parameterised dataset.![parameterizesLinked Service from github to adf](https://github.com/user-attachments/assets/b17d58d4-18a1-40c1-9796-68bcb149f019)

   - Now create sink dataset: sink -> sink dataset -> New -> Azure SQL Database -> ls_sqlDB -> table name -> ok
   - Click debug to run the pipeline. This will copy the initial data from GitHub and load it to Azure SQL table source_car_sale ![Git to SQLDb_Pipeline1_successfull](https://github.com/user-attachments/assets/b5bf45e3-d865-4791-8545-34b78e4bb810)

##6] 



