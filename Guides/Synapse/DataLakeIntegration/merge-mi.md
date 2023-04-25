# Description
This is the guide to deploy the synapse pipeline using the Managed Identity. The pipeline will make a REST call to ADME, search for ingested wellbore records from the pre-requisites earlier, flatten it, merge it with well records from the datalake on UWI and write the results back into the datalake in a new file

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

> **_NOTE:_**  Please make a note that passing parameters at the time of creating the serices is not working as expected, please <REPLACE-WITH-YOUR-VALUES> in dataset and linkedservice json files for after you download the file and then run the follwoing commands without the --parameters argument

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



