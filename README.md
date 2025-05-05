
# Car Sales Data Engineering Pipeline

## Project Overview
This project creates a data engineering pipeline using Medallion Architecture. Pipeline extracts car sales performance data from GitHub and load it to Azure SQL Database, then into ADLS bronze layer. The transformation is done using Databricks and load into ADLS silver layer. The pipeline enables incremental data loading, data transformation. Also, it creates the dimensional data model in a star schema in databricks and load it into ADLS gold layer.

## Architecture

![image](https://github.com/user-attachments/assets/30e6f8b4-cc6d-4101-8786-37356b0fa934)

The pipeline follows the Medallion Architecture, consisting of the following layers:

- Bronze Layer: Raw data extracted from GitHub and stored in ADLS Gen2 using Azure Data Factory (ADF) in Parquet file format.
- Silver Layer: Transformed data stored in ADLS Gen2, accessible by data scientists for separate analysis using Databricks.
- Gold Layer: Dimensional data model in a star schema, stored in Delta format, serving for downstream applications like Power BI.

## Tools and Technologies
- Azure Data Factory (ADF): Used for data extraction and loading.
- Azure Data Lake Storage (ADLS) Gen2: Used for storing raw and transformed data.
- Databricks: Used for data transformation and creation of the star schema.
- Azure SQL Database: Used for storing data.
- Power BI: Used for creating dashboards and visualizations.
- GitHub: Used for storing raw data.
- Parquet: Open-source file format that stores data in columns.
- Delta: Open-source data storage format that stores only the changes made to data. It's built on Parquet files.
- Star Schema: A database design that organizes data into a central table and surrounding dimension tables.
- Slowly changing dimension (SCD) type 1: A framework that overwrites existing data & inserts new data in a dimension 
  table to reflect the current state.

## Pipeline Workflow
1. Extract raw data from Azure SQL Database using ADF.
2. Store raw data in the Bronze Layer of ADLS Gen2 using ADF.
3. Transform data using Databricks and store in the Silver Layer.
4. Create a dimensional data model in a star schema and store in the Gold Layer in Delta format.
5. Use Power BI to create dashboards and visualizations.

## Incremental Loading
The pipeline is designed to store incremental data, where:

- At the first run, all data is stored.
- At subsequent runs, only incremental data is stored.

## Access and Usage
Data scientists can access the Silver Layer for separate analysis. Downstream applications like Power BI can access the Gold Layer for creating dashboards and visualizations.


# Detailed Steps:

## 1] Resource Group:
   Login to Azure portal and create one resource group which will store all the project related resources.

## 2] Data Lake:
   - Now, create one storage account where we will store our all types of data (i.e. raw, transformed and serving layer)
   - Search, "Storage Account" in azure portal and create one storage account. Fill the necessary configuration. I have selected Redundancy as LRS (Locally Redundant 
     Storage), since it is the cheapest option as it will replicate the data within same data centre. Other options are GRS (Geo Redundant Storage), ZRS (Zone 
     Reduandant Storage) and GZRS (Geo Zone Redunadant Storage)
   - In the next configuration I have enabled hierarchical namespace in order to use ADLS (Azure Data Lake Storage) instead of Blob storage. The primary difference 
     between both is we can create containers and can create multiple folders within those containers in ADLS.

     ![image](https://github.com/user-attachments/assets/be647d80-b245-48a4-90e5-87cc53698268)

   - Create three containers namely, bronze, silver and gold which defines different layers in Medallion architecture.

## 3] ADF:
   Create resource, ADF (Azure Data Factory).

## 4] Azure SQL:
   - create resource, Azure SQL.
   - Select SQL databases as the deployment option. This will help Azure to manage all admin level tasks.
   
      ![image](https://github.com/user-attachments/assets/c2778a81-a218-49ea-a920-df558e22ebeb)
     
   - SQL managed instances: will give admin level access.
   - SQL virtual machines: will provide virtual machine with Azure SQL installed in it and will have all admin level roles and responsibility.
   - SQL databses -> Enter database name -> craete new server -> Authentication Method: Use both SQL & MS Entra authentication -> set server admin login ID and 
     password -> set admin -> workload environment: development -> networking -> connectivity method -> Public Endpoints -> Review + Create
   - Now, go to resourse group and open the newly created SQL server -> networking -> Public Access -> Allow Azure services and resources to access this server.
   - At the bottom of the page, you will see databse that you have created. Open the database and click on query editor from left menu. Enter admin login ID and 
     password, you will see Azure SQL query editor.

## 5] Data Extraction:
   - Since we will be extracting data from GitHub and load it onto SQL database, we need to create one destination table (source_car_sale).
   - Go to ADF -> Author -> pipeline -> New pipeline (SourcePrep): We will extract the data stored in Github and will store it in source_car_sale table INCREMENTALLY 
     using this pipeline.
   - Go to managed tab -> linked services -> New -> HTTP -> provide GitHub base URL -> Authentication Type: Anonymous -> Test connection -> create. {This linked 
     service (ls_git)will connect ADF to GitHub}
   - Create another linked service (ls_sqlDB)to connect ADF to Azure SQL: New -> Azure SQL database -> server name -> database name -> user name -> password -> Test 
     connection -> create.
   - Go to SourcePrep pipeline -> create copy activity -> source (parameterised dataset) -> New -> HTTP -> csv -> ls_git -> Advanced -> open this dataset -> parameter        -> Name (load flag) -> connection -> Relative URL -> Dynamic Content -> paste the relative URL without file name (as file name will be dynamic I>E> changing 
     during incremental load) -> @{select parameter (load flag)} -> give the value for load flag parameter source file name...... this is how you will create 
     parameterised dataset.

     ![parameterizesLinked Service from github to adf](https://github.com/user-attachments/assets/b17d58d4-18a1-40c1-9796-68bcb149f019)

   - Now create sink dataset: sink -> sink dataset -> New -> Azure SQL Database -> ls_sqlDB -> table name -> ok
   - Click debug to run the pipeline. 
   
      ![Git to SQLDb_Pipeline1_successfull](https://github.com/user-attachments/assets/b5bf45e3-d865-4791-8545-34b78e4bb810)

   - This will copy the initial data from GitHub and load it to Azure SQL table source_car_sale.

     ![Date stored successfully into Azure SQL DB](https://github.com/user-attachments/assets/d416cb97-a4e3-4edf-8a49-098af9bb17d9)

## 6] Incremental Load Pipeline:
   - We will create new pipeline (incremental_pipeline) in which we will have 2 Lookup Activities, 1 Copy Activity & 1 Stored Procedure. 1 Lookup Activity will 
     capture last load date and another will Max date. Stored Procedure will stored the Max date and will replace the last load date from 1st lookup activity with Max 
     date using watermark table and stored procedure in SQL.

     (i) Watermark Table: This table will hold and replace Last load date value. in our dataset, min date is 'DT00001', thus we have given the last load value as day 
          before 'DT00001', i.e. "DT00000".

        ![image](https://github.com/user-attachments/assets/3af115ca-6a15-4d6f-ac50-0383414ad9db)

     (ii) Stored Procedure: Create stored procedure in SQL which will update the value of last load as Max date, everytime we run the pipeline with incremental data.

        ![image](https://github.com/user-attachments/assets/5eb232dd-214e-49c5-becb-78c645359d76)

     (iii) Lookup Activity-I (Last load date): Settings -> source dataset -> ds_sqlDB (already created) -> edit -> create parameter -> name: table name (as we are 
           defining table name in this parameter) -> Connection -> Enter manually -> add table name dynamically -> give table name value for created parameter -> 
           uncheck First row only -> use query: query ->  Query: SELECT * FROM watermark_table;
           (This will give last load date).

     ![image](https://github.com/user-attachments/assets/117a8b93-6c2b-43ae-b771-1b8b02a1204f)

     (iv) Lookup Activity-II (Max date): settings -> Source dataset: ds_sqlDB -> table name (previously created parameter): source_car_sale -> uncheck First row only             -> use query: query -> SELECT MAX(Date_ID) FROM source_car_sale;
           (This will give Max date value)

     ![image](https://github.com/user-attachments/assets/6d7ad75c-0fc0-4229-a8ba-523bf8adae28)

     (v) Copy Activity (will copy data incrementally): Source: ds_sqlDB -> table name (previously created parameter): source_car_sale -> use query: query -> query: 
         add dynamic content -> ![image](https://github.com/user-attachments/assets/7efa0de5-5604-4bc7-a2f7-6aabaf799dc3) -> Sink -> Sink dataset -> new -> ADLS gen 2 
         -> parquet -> Linked service: new -> create new linked service -> import schema: none.
           This will copy and load all the data within last load and max load date range. in our case this is initial run since we are copying and loading raw data 
           for the first time.

     ![InitialDataLoadingDetails](https://github.com/user-attachments/assets/9fe87062-e916-4275-808b-95fb0deb4db4)

     (vi) Stored Procedure Activity: This will list all the stored procedures stored within mentioned linked service or linked database.
          settings -> linked service: ls_sqlDB -> stored procedure name: UpdateWatermarkTable -> stored procedure parameters -> import -> It will automatically fetch 
          the parameter we have defined in the stored procedure query (lastload) -> value: give value of Max date from Lookup Activity-II dynamically.

     ![image](https://github.com/user-attachments/assets/5742f71c-c382-4b71-bfc6-d55c5c91a3ea)

   - We have completed our 2nd pipeline which will copy data from SQL DB and paste it into raw layer of ADLS incrementally. Click debug to run.

     ![SqlDbToADLS_Pipeline2_successsfull](https://github.com/user-attachments/assets/8ef685c6-e2fc-43f7-9516-a0d532425518)

     ![data_stored_ in_bronze_container_successfully_in_parquet_format](https://github.com/user-attachments/assets/1ef71b4c-4a9f-49c8-ae2d-f4dcced0717d)

   - We can see that the watermark table has updated the value to Max date DT01245

     ![watermarkTableSuccessfullyUpdatedValue](https://github.com/user-attachments/assets/2e798f99-b149-4139-95ae-adab66ddeea3)

   - Initial Run: In initial run last load date is (min date -1 : DT00000) and MAx date is DT01245. Copy activity will copt all the data where date range is between 
     last load date and Max date. Stored Procedure activity will store the max date value and will replace the last load date for next run as max date for initial run 
     (DT01245).

     ![image](https://github.com/user-attachments/assets/cd010cef-919d-4e1f-a832-0a02e57a64a3)

   - Incremental Run: Lets say for incremental run, we have max date as DT01247. Now in incremental run, stored procedure will replace the value of last load date to 
     DT01245. Copy activity will run in date range between DT01245 & DT01247. Stored Procedure activity will store Max date value as DT01247 and will replace last 
     load date to DT01247 for next incremental load.

     ![image](https://github.com/user-attachments/assets/7da50e74-2ad2-4aff-a185-91a9b5a55947)

## 7] Databricks:
   - Go to resource group -> search Azure databricks -> configure -> Review + Create
   - Unity Metastore: We will be attaching our databricks workspace to unity metastore to activate unity catalog.
      - Go to admin console of Databricks ([accounts.azuredatabricks.net](https://accounts.azuredatabricks.net/)) -> catalog -> create metastore (You can create only 
        one metastore in one region) -> configure -> create one dedicated container in our azure storage account and give container path in the configuration -> Give 
        access connector ID by creating Access connector for databricks.
      - Access connector for databricks: Go to azure portal -> search Access connectors for Databricks -> create -> configure -> Review + Create -> Assign a role of 
       "Storage blob data contributor" to this Access connector
      - Go to ADLS -> Access control IAM -> Add role assignment -> Storage blob data contributor -> Managed Identities (Select this to assign role to any resource)-> 
        search for newly created access connector -> Review + Assign.
      - Now give access connector ID to metastore configuration -> create
    
        ![Databricks_workspace_addedTo_metastore](https://github.com/user-attachments/assets/17a81dcd-e617-4b46-a20b-ddf404a8bcf3)

      - Attach workspace: Workspaces -> assign to workspace -> select workspace -> Assign -> Enable Unity Catalog. You can see the metastore added to the workspace
    
        ![image](https://github.com/user-attachments/assets/1cdde6f2-1ed4-41b6-895f-1d9c6fcba126)

   - Go to databricks and now create the cluster.
    
        ![Compute_Cluster_details](https://github.com/user-attachments/assets/1011c165-b179-41ef-a665-8e64dfcd0796)

   - Goto catalog tab in Databricks workspace -> create three external locations for all three containers (bronze, silver, gold) in order to read and write data in 
     databricks.
     - Catalog -> External Data -> credentials -> create credentials -> Give Access connector ID -> create -> Catalog -> External Data -> Create External 
       Location -> give path URL for ADLS container -> give newly created creds -> create -> create two more external locations.

        ![extLocation_creation_in_databricks](https://github.com/user-attachments/assets/f71de9d0-44b3-422f-b093-50f567705b7e)

## 8] Creating One Big Table in Databricks (Silver layer)
   - We will be creating one big table here using some transformation and will be storing it into Silver container.
   - Craete workspace: Workspace -> create -> Folder
   - Create notebook: Folder -> Create -> notebook -> Database_notebook
   - connect the cluster that you've created.
   - Create a catalog which is similar to database in sql. Withing catalog, create two different schemas. One for silver layer where we will be creating one big table 
     which will be used by Data Scientist/others and one for gold layer where we will keep our data model (Dimensional Data Model- Star Schema).
   - Create another notebook -> Silver_notebook : To read the data stored in the bronze layer -> Transform the data to create one big table -> Write the data in the 
     gold layer.
   - Read the data from bronze layer:

     ![image](https://github.com/user-attachments/assets/85d52b13-0808-4643-8f97-fc6c5a3ac005)

   - Transformations:

     ![image](https://github.com/user-attachments/assets/b17c96aa-1ec4-4c65-b50b-92020e71a131)

     ![transformation 1](https://github.com/user-attachments/assets/2a14e102-4331-4930-b4fa-27d74419d2e5)

     ![image](https://github.com/user-attachments/assets/6b84c621-df5a-4160-a55c-0f4db10406aa)

     Some adhoc Analysis:

     ![Adhoc analysis on agg function and visualization on top of it](https://github.com/user-attachments/assets/f31a82ef-fa7c-4a88-875d-a96444de5f81)

   - Data writing to silver layer:

     ![image](https://github.com/user-attachments/assets/43cc4179-21b8-46a5-82ec-b8d95d2b8f19)

     Data written successfully into silver layer:
     
     ![data successfully written to silver container](https://github.com/user-attachments/assets/491a23ad-f28c-4cdb-aacc-13c6db35b0a9)

   - We can also query the data, even though we have not created any table in databricks:

     ![image](https://github.com/user-attachments/assets/4116cbaf-20e1-4bfb-90d6-65e28a0674c7)

## 9] Creating a Data Model- Star Schema in Databricks (Gold layer):

![Star Schema](https://github.com/user-attachments/assets/9bd2f632-a20c-4b0b-b5ec-d89bc3feb30c)

   - Create a new notebook -> Gold_dim_model : to create first dimension
     
     (1) create a flag parameter: This will flag if its initial run or incremental run:

     ![Incremental flag creation with dewfalt value as 0](https://github.com/user-attachments/assets/755f2f25-7991-4753-ba3a-b83d9647354d)

     (2) Create dataframe df_src which will read the model dimensions from silver layer:

     ![image](https://github.com/user-attachments/assets/cc7175e5-edab-4fcf-a8f3-e3abff595f8c)

     (3) Create df_sink which will bring blank schema in case of initial run and will bring all the existing data in case of incremental run:

     ![image](https://github.com/user-attachments/assets/b724a9cc-af77-4391-bc1e-16d69a6685d2)

     (4) Now, apply left join between df_src & df_sink to filter old and new records:

     ![left Join  Initial Load-all null so we can insert all the data](https://github.com/user-attachments/assets/71e40036-7dcb-47d0-b0eb-d8ed278b6804)

     (5) create df_filter_old which will have all the records with not null value in dim_model_key column: (We will update this data since it has updated info against 
         existing dim_model_key)

     ![df_filter_old](https://github.com/user-attachments/assets/f78955c9-4b4d-4755-9d25-792fae3011e6)

     (6) create df_filter_new which will have all the records with null value in dim_model_key column: (We will insert this data)

     ![df_filetr_new-without key col as we are goint to create our own surr key col](https://github.com/user-attachments/assets/00e147cd-8a95-426b-84e7-48b1b0c5aede)

     (7) Create surrogate key column and add max surrogate key using incremental_flag:

     ![max surrogate key](https://github.com/user-attachments/assets/21c7d4bc-c341-4e4b-8422-d6189c1f35f0)

     ![create surrogate key column and add max to it](https://github.com/user-attachments/assets/2ef5ee14-07f3-408a-b3f1-5002bbcb8628)

     (8) Create df_final which is df_filter_old + df_filter_new:

     ![df_final](https://github.com/user-attachments/assets/ebb4f862-97d1-46b4-a02a-6cb74da0d2ad)

     (9) Now, we will apply slowly changing dimension logic:

     ![SCD T-1 (Upsert) and table created](https://github.com/user-attachments/assets/80d9f3f1-d98f-422f-a9be-f5820a319b47)

     Table is now created after initial run and our 1st scd is ready:

     ![1st slowly changing dimension table ready (dim_model)](https://github.com/user-attachments/assets/01bf44bc-1bdb-460d-88cf-cfc8599bb60d)

     (10) Similarly, remaining dimension tables are created:

     ![All dimension tables are created](https://github.com/user-attachments/assets/7b2ec884-d725-4248-945e-0dd9611a7c0e)


   - Create a notebook -> Gold_fact_sales (to create fact table)
     
     (1) Create df_silver to read the silver layered data in order to see which columns we need to include in fact table:

     ![reading silver data for fact table](https://github.com/user-attachments/assets/713f64f9-6dda-4874-86a9-cef22cb90b23)

     (2) Create df for all the dimensions:

     ![Reading all the dimension tables](https://github.com/user-attachments/assets/5e00878a-746a-4e41-b662-8071e909c124)

     (3) Apply join between df_silver and all the df for dimensions in order to bring surrogate keys:

     ![Bringing all the dim keys to the fact table](https://github.com/user-attachments/assets/011e1027-5803-437e-bde8-57b6cba37901)

     (4) Write the data:

     ![writing fact table to gold layer with UPSERT](https://github.com/user-attachments/assets/3b59c38c-42be-4a79-81eb-27d8b2272dc6)

     Fact table created:

     ![Fact table created](https://github.com/user-attachments/assets/cb198352-2e52-40bc-b647-1aa777603be3)

     ![Fact sales can be seen in catalog](https://github.com/user-attachments/assets/c00b257c-421a-471a-9ec0-9c7cea8fc773)


## 10] Running pipeline in Databricks:

   - We can create and run the notebook pipeline in databricks itself using workflow:

     ![successful run in Databricks workflow](https://github.com/user-attachments/assets/c7eb2068-690a-49cb-b7cd-fd4c1972ae6a)

     ![workflow timeline](https://github.com/user-attachments/assets/6ea74770-498a-43c5-b186-642fa95d8b3a)

## 11] SQL editor in Databricks:

   - Since our data model is now ready, downstream user like data analyst/data scientist can use the SQL editor in Databricks to query the data.

     ![data queried using SQL editor in databricks](https://github.com/user-attachments/assets/7bcdea1b-7c82-4b3f-82bc-d344404feb31)

     
## 12] Incremental data load:

   - Now, go to ADF to pull the incremental data -> SourcePrep pipeline -> give the incremental file name (IncrementalSales.csv) -> run the pipeline (This will load 
     the incremental data to Azure SQL) ![Change the input file name in source pipeline](https://github.com/user-attachments/assets/40675f01-e794-4a6c-b3da-32943ccb6185)
     -> go to incremental_pipeline -> run the pipeline -> This should load the data incrementally into bronze layer.

     ![Increm pipeline ran successfully in ADF](https://github.com/user-attachments/assets/4683c57d-9ef3-42be-a115-9f021af6d179)

   - We can check if our data is loaded incrementally or not:

     Go to sql query editor in Azure. query the watermark table. It should have updated last load date:

     ![watermark table has now updated value ](https://github.com/user-attachments/assets/658dcd84-c0f2-454b-a28c-d2d6cd68f727)

     
   - Go to Azure Databricks -> Run the workflow created for incremental run.

     ![Incremental run was successful in databricks workflow](https://github.com/user-attachments/assets/9d9b94cd-6de6-4874-b57d-a09a4f12b0ee)

     Workflow ran successfully!


   - We can validate the incremantally loaded data:

       (1) Go to SQL Editor in Azure Databricks -> Query the fact table

     ![Increm Data Validation1](https://github.com/user-attachments/assets/bb9a9e0a-b15d-493b-8f83-1d9e24a100ae)

       (2) Query dim_dealer table:

     ![Increm data validation 2](https://github.com/user-attachments/assets/5ede36bc-fefb-41c6-9ebd-d3674f5c5554)

       (3) Query dim_model table:

     ![new rec added](https://github.com/user-attachments/assets/09ed46ea-c420-479d-85d2-c1cbc6b2e716)


## 13] End to end pipeline in ADF:

   - We can also run the full pipeline in ADF itself. To add the Databricks Notebook -> add notebook activity -> create linked service -> To create access token -> go 
     to databricks -> profile -> settings -> developer -> access token -> manage -> Generate new -> generate -> copy and paste it in the linked service creation 
     dashboard -> create linked service -> settings -> Give Notebook path

   - Similarly add all attach all the notebooks and run the pipeline:

     ![end to end pipeline in ADF](https://github.com/user-attachments/assets/57090e4c-b9e9-4f05-96e3-9904fee23091)


## 14] PowerBI connection:

   Since we have created our data warehouse we can connect it to visualization tool eg. PowerBI:

   Azure Databricks -> Partner connect -> Search Microsoft Power BI -> download connection file -> Open the downloaded file with Power BI application.

   All the tables will be there in the tool. We can build visualization on top of it. 


#Project Summary:

![diagram](https://github.com/user-attachments/assets/568deb19-d1cc-4ef5-b44e-ad2350f4dca7)


     

     


       

     


     

     



     




     


















     


     

      





          






