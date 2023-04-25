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

## [Using Synapse Managed Identity](/Guides/Synapse/DataLakeIntegration/merge-mi.md)