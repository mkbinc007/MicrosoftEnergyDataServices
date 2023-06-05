# Description
The intent with the solution is to showcase how we can merge data from a datalake backed by ADLSGen2 and an ADME instance using a Synapse pipeline. In this tutorial we are specifically, looking at a sample wells data from the datalake and pulling wellbore data from ADME, flattening the data, merging the two on UWI (Universal Well Identifier) and writing the merged data back to the datalake.
<br />


# Shared Prerequisites
Follow these prerequisites for successful deployment of the solution. We will be using combination of Azure CLI and Azure Portal for the solution
<details>
<summary>1. Log into your Azure Subscription</summary>

Download from [aka.ms/azurecli](https://aka.ms/azurecli).  
Login to the Azure CLI using the command below, and your user with subscription owner rights:
```Powershell
az login
```
Verify that the right subscription is selected:
```Powershell
az account show
```
If the correct subscription is not selected, run the following command:
```Powershell
az account set --subscription %subscription_id%
```
</details>

<details>
<summary>2. Register Azure Resource Providers</summary>

```Powershell
az provider register --namespace Microsoft.DataFactory
az provider register --namespace Microsoft.DataLakeStore
az provider register --namespace Microsoft.OpenEnergyPlatform
az provider register --namespace Microsoft.Sql
az provider register --namespace Microsoft.Storage
az provider register --namespace Microsoft.Synapse
```
</details>

<details>
<summary>3. Create an Azure Resource Group</summary>

```Powershell
az group create `
    --name <resource-group> `
    --location <location>
```
</details>

<details>
<summary>4. Create ADLSGen2 Storage Account and the required Container/FileSystem. Make a note of the STORAGE_ACCOUNT_NAME you use, it is required during the next part of the tutorial to create a linked service</summary>

```Powershell
az storage account create `
    --name <storage-account> `
    --resource-group <resource-group> `
    --sku Standard_LRS `
    --hns true
    --location <location>
```

<details open>
<summary>Then create a container also referred to as filesystem to use as the source/destination to read/write files to</summary>

```Powershell
az storage container create `
    --account-name <storage-account> `
    --name <container> `
```
</details>

<details open>
<summary>Download the [wells sample csv](Guides/Synapse/DataLakeIntegration/sample_data/well_mahesh.csv) to your local machine and then upload the file to the newly created container</summary>

```Powershell
az storage blob upload `
    --account-name <storage-account> `
    --container-name <container> `
    --name wells_mahesh.csv `
    --file @<path-to>/wells_mahesh.csv
```
</details>
</details>

<details>
<summary>5. Create Azure Synapse Workspace</summary>

```Powershell
az synapse workspace create `
    --name <workspace-name> `
    --file-system <filesystem> `
    --resource-group <resource-group> `
    --storage-account <storage-account> `
    --sql-admin-login-user <sql-admin-username> `
    --sql-admin-login-password <sql-admin-password> `
    --location <location>
```

<details open>
<summary>Open Synapse Workspace for public access</summary>

```Powershell
az synapse workspace firewall-rule create `
    --name allowAll `
    --resource-group <resource-group> `
    --workspace-name <workspace-name> `
    --start-ip-address 0.0.0.0 `
    --end-ip-address 255.255.255.255
```
</details>
</details>

<details>
<summary>6. Create an instance of Azure Data Manager for Energy (ADME)</summary>

Please follow the instructions at [create ADME public preview instance](https://learn.microsoft.com/en-us/azure/energy-data-services/quickstart-create-microsoft-energy-data-services-instance). Please make a note of the ADME_APPREG_APP_ID, DATA_PARTITION_ID and the ADME_HOST_NAME that you'd have used during the creation of ADME instance. It is also recommended at this time, to use the same azure region for all the services you create.
</details>

<details>
<summary>7. Ingest sample CSV wellbore data into ADME</summary>

Please follow all the instructions at [steps to perform CSV parser ingestion](https://learn.microsoft.com/en-us/azure/energy-data-services/tutorial-csv-ingestion) to ingest sample CSV that has 4 ficticious wellbore records. This part of the tutorial you will need Postman to make the necessary API calls to ADME to ingest the sample data.

> **_NOTE:_**  Please make sure that step#2 and step#6 from the ingestion tutorial are followed properly per instructions below.

During step #2 (create a schema) of the [ingest CSV tutorial](https://learn.microsoft.com/en-us/azure/energy-data-services/tutorial-csv-ingestion), please capture the value of "id" from the response and save it, this will be the SEARCH_QUERY that we will use when making a call to ADME later to get the ingested records.

During step #6 (pointing to the csv) of the [ingest CSV tutorial](https://learn.microsoft.com/en-us/azure/energy-data-services/tutorial-csv-ingestion), please use the wellbore.csv from this [location](/Guides/Synapse/DataLakeIntegration/sample_data/wellbore.csv) instead of the one listed.
</details><br />

We will be using Managed Identity of the provisioned Synapse Workspace as an authentication mechanism into ADME. Please make sure that all of the above prerequisites are complete and you have all the required information captured for the rest of the tutorial. Please click on the link below.

## Using Synapse & Dataflows for merge

This part of the guide is to deploy the synapse pipeline using the Managed Identity. The pipeline will make a REST call to ADME, search for ingested wellbore records from the pre-requisites earlier, flatten it, merge it with well records from the datalake on UWI and write the results back into the datalake in a new file

### 1. Grant Synapse Workspace access to make an API call to ADME
1. Obtain an Access Token for a user with access to write to the Microsoft Energy Data Services Entitlements service. For more information see [learn.microsoft.com manage users](https://learn.microsoft.com/en-us/azure/energy-data-services/how-to-manage-users)
2. Get the Synapse Workspace Managed Identity ObjectID, and then lookup the Application ID which will be used in the next API call. Make a note of the Synape-MI-AppID
    ```Powershell
    $synapsemi = (az synapse workspace show ` 
        --name <synapse-workspace> `
        --resource-group <resource-group> | `
        ConvertFrom-Json).identity.principalId
    (az ad sp show --id $synapsemi | ConvertFrom-Json).appId
    ```

3. Run the below REST API call through Postman or other API tool to add the Synapse Workspace Managed Identity ObjectID to the users AND users.datalake.editors groups (which is required for any access to ADME). You will be making two calls. You will use the synapsemi from the above when replacing the <Synapse-MI-AppID> below
    ```Powershell
    curl --location --request POST 'https://<instance>.energy.azure.com/api/entitlements/v2/groups/users@<data-partition-id>.dataservices.energy/members' `
        --header 'data-partition-id: <data-partition-id>' `
        --header 'Authorization: Bearer <access_token>' `
        --header 'Content-Type: application/json' `
        --data-raw '{
                        "email": "<Synapse-MI-AppID>",
                        "role": "MEMBER"
                    }'
    ```

    ```Powershell
    curl --location --request POST 'https://<instance>.energy.azure.com/api/entitlements/v2/groups/users.datalake.editors@<data-partition-id>.dataservices.energy/members' `
        --header 'data-partition-id: <data-partition-id>' `
        --header 'Authorization: Bearer <access_token>' `
        --header 'Content-Type: application/json' `
        --data-raw '{
                        "email": "<Synapse-MI-AppID>",
                        "role": "MEMBER"
                    }'
    ```

    Make sure that your response from the above calls is something like this
        <details>
        <summary>Sample Response</summary>

        ```JSON
        HTTP/1.1 200 OK
        Date: Mon, 17 Apr 2023 12:11:41 GMT
        Content-Type: application/json

        {
        "email": "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee",
        "role": "MEMBER"
        }
        ```
        </details>

### 2. Grant Synapse Workspace access to read data from source ADLS

We will give the Synapse Workspace Managed identity access as Storage Blob Data Reader and Storage Blob Data Contributor to the Storage Account we've specified. 
```Powershell
$synapsemi = (az synapse workspace show --name <synapse-workspace> --resource-group <resource-group> | ConvertFrom-Json).identity.principalId
az role assignment create --assignee $synapsemi --role 'Storage Blob Data Reader' --scope /subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.Storage/storageAccounts/<storage-account>
```

```Powershell
$synapsemi = (az synapse workspace show --name <synapse-workspace> --resource-group <resource-group> | ConvertFrom-Json).identity.principalId
az role assignment create --assignee $synapsemi --role 'Storage Blob Data Contributor' --scope /subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.Storage/storageAccounts/<storage-account>
```

### 3. Create Linked Services and Datasets
We will create all the required linked services and the datasets needed for the pipeline

> **_NOTE:_**  Please make a note that passing parameters at the time of creating the serices is not working as expected, please REPLACE-WITH-YOUR-VALUES where you see <> in dataset and linkedservice json files after you download the file and then run the following commands without the --parameters argument for now, will publish the parameterization later.

1. Download the [linkedservice_datalake.json](src/linkedservice_datalake.json) to your local machine.
   1. Run the following command and refer to the downloaded file to create the linked service, replace the <STORAGE_ACCOUT_NAME> with your account name that you used to create during the pre-requisites
    ```Powershell
    az synapse linked-service create `
        --name <linked-service-name> `
        --workspace-name <synapse-workspace> `
        --parameters storageAccountName=<STORAGE_ACCOUNT_NAME>
        --file @<path-to>/linkedservice_datalake.json
    ```
2. Download the [linked_service_adme_rest.json](src/linked_service_adme_rest.json) to your local machine
   1. Run the following command and refer to the downloaded file to create the linked service. Substitute the values of <ADME_HOST_NAME> and the <ADME_APPREG_APP_ID> with relevent values, you captured during pre-requisites
    ```Powershell
    az synapse linked-service create `
        --name <linked-service-name> `
        --workspace-name <synapse-workspace> `
        --parameters `
            ADME_HOST_NAME=<ADME_HOST_NAME> `
            ADME_APPREG_APP_ID=<ADME_APPREG_APP_ID> `
        --file @<path-to>/linked_service_adme_rest.json
    ```
3. Download the [dataset_adme_source_rest.json](src/dataset_adme_source_rest.json) to your local machine
   1. Run the following command and refer to the downloaded file to create the dataset
    ```Powershell
    az synapse dataset create `
        --name <dataset-name> `
        --workspace-name <synapse-workspace> `
        --file @<path-to>/dataset_adme_source_rest.json
    ```
4. Download the [dataset_adme_sink_lake.json](src/dataset_adme_sink_lake.json) to your local machine
   1. Run the following command and refer to the downloaded file to create the dataset
    ```Powershell
    az synapse dataset create `
        --name <dataset-name> `
        --workspace-name <synapse-workspace> `
        --file @<path-to>/dataset_adme_sink_lake.json.json
    ```
5. Download the [dataset_dataflow_source_json.json](src/dataset_dataflow_source_json.json) to your local machine
   1. Run the following command and refer to the downloaded file to create the dataset
    ```Powershell
    az synapse dataset create `
        --name <dataset-name> `
        --workspace-name <synapse-workspace> `
        --file @<path-to>/dataset_dataflow_source_json.json
    ```    
6. Download the [dataset_dataflow_source_wells.json](src/dataset_dataflow_source_wells.json) to your local machine
   1. Run the following command and refer to the downloaded file to create the dataset
    ```Powershell
    az synapse dataset create `
        --name <dataset-name> `
        --workspace-name <synapse-workspace> `
        --file @<path-to>/dataset_dataflow_source_wells.json
    ```    
7. Download the [dataset_sink_final_combined.json](src/dataset_sink_final_combined.json) to your local machine
   1. Run the following command and refer to the downloaded file to create the dataset
    ```Powershell
    az synapse dataset create `
        --name <dataset-name> `
        --workspace-name <synapse-workspace> `
        --file @<path-to>/dataset_sink_final_combined.json
    ```
8. Verify that all the datasets and linked services are created successfully


### 4. Create the Dataflow
We will create a dataflow actvity that will be referenced in the pipeline that has the merge logic basesd on UWI
Download the [dataflow_join.json](src/dataflow_join.json) to your local machine. Run the following command and refer to the downloaded file to create the dataflow
```Powershell
    az synapse data-flow create `
        --name <dataflow-name> `
        --workspace-name <synapse-workspace> `
        --file @<path-to>/dataflow_join.json
```    


### 5. Create the Synapse Pipeline
Create the pipeline that binds all this together and ready for test run
Dodwnload tge [pipeline_adme_lake.json](src/pipeline_adme_lake.json) to your local machine. Run the following command and refer to the downloaded file to create the pipeline. Replace the <DATA_PARTITION_ID> and <SEARCH_QUERY> with values you captured during the pre-requisites
```Powershell
    az synapse pipeline create `
        --name <pipeline-name> `
        --workspace-name <synapse-workspace> `
        --parameters `
            DATA_PARTITION_ID=<DATA_PARTITION_ID> `
            SEARCH_QUERY=<SEARCH_QUERY> `
        --file @<path-to>/pipeline_adme_lake.json
```


### 6. Test the Pipeline
1. Log into [Azure Portal](https://portal.azure.com) and navigate to the resourceGroup-->synapseWorkspace-->synapseStudio, ensure that all the linked services, datasets, dataflow and the pipeline are created successfully
2. Navigate to pipeline section and click on pipeline 'pipeline_adme_lake'
3. Click on 'Add trigger' button on the top and then click 'Trigger now'
4. Go to the 'Monitor' section of the synapse studio and monitor the run of the pipeline
5. After succesful run of the pipeline you should see a wells_weebore_combined.csv in your storage account that you provisioned. This CSV file has been created as part of the run where it joined wells data (from storage) and wellbore data (from ADME) on UWI field. 