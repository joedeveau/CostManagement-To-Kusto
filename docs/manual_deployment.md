# Exporting Azure Cost Management Data to ADX


## Process Overview

At a high-level the process will be:

1.  Daily export your cost data using Azure Cost Management to blob
    storage

2.  When its created, load the blob automatically into ADX

3.  Ensure you don't introduce duplicate records with your ingestion

Eliminating duplicates is the most important part of this process. To understand why deduplication is important, you need to understand how Azure Cost Management export [works](https://learn.microsoft.com/azure/cost-management-billing/costs/tutorial-export-acm-data).

## Azure Cost Management Export Overview

With Azure Cost Management Exports, you can export your data daily. The export places a new csv file on azure blob storage daily with all the data upto that day, month-to-date, or weekly depending on your selection. For example, on the 1^st^ of the month you'll have a csv file with one day's worth of data, and on the 20^th^ day of the month you'll have another csv file with 20 days-worth of data. However, these costs aren't finalized until the end of the month, so your 1^st^ day of the month may not match if you compare the exports of the 20^th^ day for the 1^st^ day expenditures. Therefore, if we do not handle duplicate records correctly the data will NOT make sense.

1.  We can end up with a lot of duplicates for each day.

2.  We end up keeping the wrong record instead of the latest.

3.  An update to a record made later during the month doesn't appear (ie. missing data).

## Example Implementation

For this implementation the following are needed:

1.  Azure Data Explorer (required for performance needs)

2.  Azure Storage Account

3.  Azure Data Factory (allows to easily tag extents)

4.  Cost Management Export

We will walk through the setup, but you can leverage the [automated deployment](https://github.com/wpbrown/azmeta-pipeline) method created by Will Brown.

You may skip ahead to section. Verify that ADF has blob reader access to the storage location, the cost management export is enabled and the ADF pipeline is published.

### Step 1: Setting up ADX

This setup will contain the following in ADX

1.  UsagePreliminaryIngest: Raw table where data is ingested (bronze)

2.  UsagePreliminaryMapping: Raw table CSV mapping for data ingestion

3.  UsagePreliminary: Curated table (silver) used for querying & reporting

4.  IngestToUsagePreliminary: KQL function that does the minor transformation of the raw table and used by the update policy of the curated table

5.  Update policy on table UsagePreliminary so data automatically flows transactionally from raw table *UsagePreliminaryIngest* into landing table *UsagePreliminary*

6.  Retention policy on the raw table *UsagePreliminaryIngest* with a retention of 0, eliminating ADX storage costs, given data is transformed into *UsagePreliminary* immediately upon ingestion.

The below script can be utilized to create everything with the correct
schema.

```
.execute database script <|
//Create raw table for usage data
//
.create table UsagePreliminaryIngest (InvoiceSectionName: string,
AccountName: string, AccountOwnerId: string, SubscriptionId: string,
SubscriptionName: string, ResourceGroup: string, ResourceLocation:
string, Date: datetime, ProductName: string, MeterCategory: string,
MeterSubCategory: string, MeterId: string, MeterName: string,
MeterRegion: string, UnitOfMeasure: string, Quantity: decimal,
EffectivePrice: decimal, CostInBillingCurrency: decimal, CostCenter:
string, ConsumedService: string, ResourceId: string, Tags: string,
OfferId: string, AdditionalInfo: dynamic, ServiceInfo1: string,
ServiceInfo2: string, ResourceName: string, ReservationId: string,
ReservationName: string, UnitPrice: decimal, ProductOrderId: string,
ProductOrderName: string, Term: string, PublisherType: string,
PublisherName: string, ChargeType: string, Frequency: string,
PricingModel: string, AvailabilityZone: string, BillingAccountId:
string, BillingAccountName: string, BillingCurrencyCode: string,
BillingPeriodStartDate: datetime, BillingPeriodEndDate: datetime,
BillingProfileId: string, BillingProfileName: string, InvoiceSectionId:
string, IsAzureCreditEligible: string, PartNumber: string, PayGPrice:
decimal, PlanName: string, ServiceFamily: string)
//
//Create the ingestion mapping
//
.create-or-alter table Usage ingestion csv mapping
'UsagePreliminaryMapping'
'[{"Name":"InvoiceSectionName","DataType":"","Ordinal":"0","ConstValue":null},{"Name":"AccountName","DataType":"","Ordinal":"1","ConstValue":null},{"Name":"AccountOwnerId","DataType":"","Ordinal":"2","ConstValue":null},{"Name":"SubscriptionId","DataType":"","Ordinal":"3","ConstValue":null},{"Name":"SubscriptionName","DataType":"","Ordinal":"4","ConstValue":null},{"Name":"ResourceGroup","DataType":"","Ordinal":"5","ConstValue":null},{"Name":"ResourceLocation","DataType":"","Ordinal":"6","ConstValue":null},{"Name":"Date","DataType":"","Ordinal":"7","ConstValue":null},{"Name":"ProductName","DataType":"","Ordinal":"8","ConstValue":null},{"Name":"MeterCategory","DataType":"","Ordinal":"9","ConstValue":null},{"Name":"MeterSubCategory","DataType":"","Ordinal":"10","ConstValue":null},{"Name":"MeterId","DataType":"","Ordinal":"11","ConstValue":null},{"Name":"MeterName","DataType":"","Ordinal":"12","ConstValue":null},{"Name":"MeterRegion","DataType":"","Ordinal":"13","ConstValue":null},{"Name":"UnitOfMeasure","DataType":"","Ordinal":"14","ConstValue":null},{"Name":"Quantity","DataType":"","Ordinal":"15","ConstValue":null},{"Name":"EffectivePrice","DataType":"","Ordinal":"16","ConstValue":null},{"Name":"CostInBillingCurrency","DataType":"","Ordinal":"17","ConstValue":null},{"Name":"CostCenter","DataType":"","Ordinal":"18","ConstValue":null},{"Name":"ConsumedService","DataType":"","Ordinal":"19","ConstValue":null},{"Name":"ResourceId","DataType":"","Ordinal":"20","ConstValue":null},{"Name":"Tags","DataType":"","Ordinal":"21","ConstValue":null},{"Name":"OfferId","DataType":"","Ordinal":"22","ConstValue":null},{"Name":"AdditionalInfo","DataType":"","Ordinal":"23","ConstValue":null},{"Name":"ServiceInfo1","DataType":"","Ordinal":"24","ConstValue":null},{"Name":"ServiceInfo2","DataType":"","Ordinal":"25","ConstValue":null},{"Name":"ResourceName","DataType":"","Ordinal":"26","ConstValue":null},{"Name":"ReservationId","DataType":"","Ordinal":"27","ConstValue":null},{"Name":"ReservationName","DataType":"","Ordinal":"28","ConstValue":null},{"Name":"UnitPrice","DataType":"","Ordinal":"29","ConstValue":null},{"Name":"ProductOrderId","DataType":"","Ordinal":"30","ConstValue":null},{"Name":"ProductOrderName","DataType":"","Ordinal":"31","ConstValue":null},{"Name":"Term","DataType":"","Ordinal":"32","ConstValue":null},{"Name":"PublisherType","DataType":"","Ordinal":"33","ConstValue":null},{"Name":"PublisherName","DataType":"","Ordinal":"34","ConstValue":null},{"Name":"ChargeType","DataType":"","Ordinal":"35","ConstValue":null},{"Name":"Frequency","DataType":"","Ordinal":"36","ConstValue":null},{"Name":"PricingModel","DataType":"","Ordinal":"37","ConstValue":null},{"Name":"AvailabilityZone","DataType":"","Ordinal":"38","ConstValue":null},{"Name":"BillingAccountId","DataType":"","Ordinal":"39","ConstValue":null},{"Name":"BillingAccountName","DataType":"","Ordinal":"40","ConstValue":null},{"Name":"BillingCurrencyCode","DataType":"","Ordinal":"41","ConstValue":null},{"Name":"BillingPeriodStartDate","DataType":"","Ordinal":"42","ConstValue":null},{"Name":"BillingPeriodEndDate","DataType":"","Ordinal":"43","ConstValue":null},{"Name":"BillingProfileId","DataType":"","Ordinal":"44","ConstValue":null},{"Name":"BillingProfileName","DataType":"","Ordinal":"45","ConstValue":null},{"Name":"InvoiceSectionId","DataType":"","Ordinal":"46","ConstValue":null},{"Name":"IsAzureCreditEligible","DataType":"","Ordinal":"47","ConstValue":null},{"Name":"PartNumber","DataType":"","Ordinal":"48","ConstValue":null},{"Name":"PayGPrice","DataType":"","Ordinal":"49","ConstValue":null},{"Name":"PlanName","DataType":"","Ordinal":"50","ConstValue":null},{"Name":"ServiceFamily","DataType":"","Ordinal":"51","ConstValue":null}]'
//
// Create the landing table for the data. This is the table that will
hold the data and you will query
//
.create table UsagePreliminary (InvoiceSectionName: string, AccountName:
string, AccountOwnerId: string, SubscriptionId: string,
SubscriptionName: string, ResourceGroup: string, ResourceLocation:
string, Date: datetime, ProductName: string, MeterCategory: string,
MeterSubCategory: string, MeterId: string, MeterName: string,
MeterRegion: string, UnitOfMeasure: string, Quantity: decimal,
EffectivePrice: decimal, CostInBillingCurrency: decimal, CostCenter:
string, ConsumedService: string, ResourceId: string, Tags: dynamic,
OfferId: string, AdditionalInfo: dynamic, ServiceInfo1: string,
ServiceInfo2: string, ResourceName: string, ReservationId: string,
ReservationName: string, UnitPrice: decimal, ProductOrderId: string,
ProductOrderName: string, Term: string, PublisherType: string,
PublisherName: string, ChargeType: string, Frequency: string,
PricingModel: string, AvailabilityZone: string, BillingAccountId:
string, BillingAccountName: string, BillingCurrencyCode: string,
BillingPeriodStartDate: datetime, BillingPeriodEndDate: datetime,
BillingProfileId: string, BillingProfileName: string, InvoiceSectionId:
string, IsAzureCreditEligible: string, PartNumber: string, PayGPrice:
decimal, PlanName: string, ServiceFamily: string)
//
//Create the function that will do the minor transformation from the raw
table
//
.create-or-alter function  IngestToUsagePreliminary() {
    UsagePreliminaryIngest
    | extend Tags = todynamic(strcat('{', Tags, '}')), ResourceId=tolower(ResourceId)
}
//
// Create the update policy on the landing table so data will flow from
raw table to landing table
//
.alter table UsagePreliminary policy update ```
[
  {
    "IsEnabled": true,
    "Source": "UsagePreliminaryIngest",
    "Query": "IngestToUsagePreliminary()",
    "IsTransactional": true,
    "PropagateIngestionProperties": true,
    "ManagedIdentity": null
  }
] ```


//
// Set the retention on the landing table to 0 days
//
.alter table UsagePreliminaryIngest policy retention ```
{
   "SoftDeletePeriod": "00:00:00",
   "Recoverability": "Disabled"
} ```

``````

### Step 2: Storage Account Container

On your blob storage account create a container called
"usage-preliminary". This is where you will export the cost management
data. 

![img](/images/manual_deployment1.png)

### Step 3: Grant the ADF System Assigned Identity Permission

1.  We are going to use the ADF System Assigned Identity to read the blob from our storage account and to write the results to ADX. Therefore, we need to make sure it has Storage Blob Data Reader RBAC permissions on this container.

![img](/images/manual_deployment2.png)

2.  Grant the ADF System Assigned Identity Database Admin permission on your ADX Database. This is required because we are dropping extents and need more permission than just ingestor.

![img](/images/manual_deployment3.png)

### Step 4: Create the Azure Cost Management Export

At this point, I recommend creating the Azure Cost Management Export, so that we'll have a csv file in storage to author the ADF pipeline mapping. If this is your first time working with an export in Azure Cost Management check out the article [here](https://learn.microsoft.com/en-us/azure/cost-management-billing/costs/tutorial-export-acm-data?tabs=azure-portal).

Below you can see how we configure the Cost Management Export which will affect the values in the ADF pipeline:

![img](/images/manual_deployment4.png)

Next, click Run now to trigger the export and generate a CSV file in the storage account, which we'll to utilize during the creation of the ADF Pipeline.

![img](/images/manual_deployment5.png)

### Step 5: Create your Linked Services in Azure Data Factory

Create two linked services in ADF. One to the storage account and one to ADX (Kusto). If you haven't done this in the past, please read the
[Linked Service with UI](https://learn.microsoft.com/azure/data-factory/concepts-linked-services?tabs=data-factory#linked-service-with-ui) and [Management Hub](https://learn.microsoft.com/azure/data-factory/author-management-hub) articles.

You should end up with the following two Linked Services:

![img](/images/manual_deployment6.png)

Both linked services are using the ADF System Assigned Identity

![img](/images/manual_deployment7.png)

![img](/images/manual_deployment8.png)

### Step 6: Create the ADF Pipeline

Create a new Pipeline in ADF called "ingest_usage_preliminary". You may copy the pipeline's JSON definition from Will Brown's repo or proceed defining it manually using the following instructions.

1.  Add two pipeline Parameters of type String (BlobPath and BlobName)

![img](/images/manual_deployment9.png)

2.  Add a "Copy data" activity from the Activities pane on the left, expand "Move and Transform" section, or type Copy data into the search and double click it or drag it onto the blank canvas. I've set the "General" tab as shown:

![img](/images/manual_deployment10.png)

3.  Click the **Source** tab of this activity, to create a new dataset. Choose Azure Blob Storage and CSV. Configure the dataset properties:

![img](/images/manual_deployment11.png)

4.  Open the Source dataset and set the **Escape character** to be double quote (")

![img](/images/manual_deployment12.png)

![img](/images/manual_deployment13.png)

5.  Next, set the **File path type** to **Prefix** and click Add dynamic content

![img](/images/manual_deployment14.png)

6.  In the popup box paste the following text:

@{concat(pipeline().parameters.BlobPath, '/',
pipeline().parameters.BlobName)}

7.  Next, lets configure the Sink tab of the Copy Data activity:
- Sync dataset: Select the ADX linked service from the drop down
- Table: "UsagePreliminaryIngest"
- Ingestion mapping name: "UsagePreliminaryMapping"
- Additional properties
    - click **Add dynamic content** and paste in the following text:

```
@concat('{"tags":"["',last(split(pipeline().parameters.BlobPath, '/')),'","drop-by:', pipeline().RunId, '"]"}')
```

*Note, this effectively adds a custom tag to our data once ingested into ADX (Kusto). The extents will be tagged a concatenated string of the BlobPath (month of the export) and the key-value "drop-by:" plus the ADF pipeline runid (unique id) which ingested the data.*

8.  Next, click on the "Mapping" tab of our Copy Data activity. Choose the option to "Import schemas". Use the exported CSV created in Step 4 and paste in the BlobPath and BlobName

![img](/images/manual_deployment15.png)

9.  Verify that the mapping comes across as you'd expect

![img](/images/manual_deployment16.png)

10. Next, under the Activities pane on the left, add the "Azure Data Explorer Command" activity by typing it into the search and double clicking it or drag it to the canvas. It should be under the "Azure Data Explorer" Activities section. Have it execute on "Success" after the "Copy Data" activity on the canvas.

![img](/images/manual_deployment17.png)

11. Under the General Tab give the activity a name

![img](/images/manual_deployment18.png)

12. In the "Connection" tab select your ADX linked service

![img](/images/manual_deployment19.png)

13. On the "Command" tab click **Add dynamic content** and paste the
    following text:
```
@concat('.drop extents <|
.show table UsagePreliminary extents
| extend Tags = split(Tags, "rn")
| where set_has_element(Tags, "', last(split(pipeline().parameters.BlobPath, '/')),'") and not(set_has_element(Tags, "drop-by:', pipeline().RunId,'"))')
```

*Note, this will run the ".drop extents" command but only drop extents for the same month that doesn't have the pipeline().RunId of the current
run.*

Below is what the Command dynamic content should look like.

![img](/images/manual_deployment20.png)

14. Next, add the trigger to schedule our pipeline. Click on "Add trigger" and choose "New/Edit".

![img](/images/manual_deployment21.png)

15. Create a new "BlobEventsTrigger" as shown:

![img](/images/manual_deployment22.png)

16. Click next until you get to the Trigger Run Parameters
-   BlobPath: @replace(trigger().outputs.body.folderPath,    'usage-preliminary/', '')
-   BlobName: @trigger().outputs.body.fileName

17. Click **Save** and **Publish all** of your changes

![img](/images/manual_deployment23.png)

### Testing

Now it's time to test and make sure everything works as expected. Luckily this is pretty easy to test.

1.  Got back to your export and execute it manually just like in Step 4

2.  This should automatically trigger the ADF Pipeline. In ADF studio, click Monitoring on the left menu blade, then Pipeline runs, and you should see the newly created pipeline has been Triggered. Verify it ran and if it failed, you can click on it to see why.

![img](/images/manual_deployment24.png)

3.  If the pipeline ran successfully, proceed to your ADX query url, select your database and verify the data in the curated table shows up as expected. Below I simply took 10 records from the *UsagePreliminary* table.

![img](/images/manual_deployment25.png)

4.  For the logic to prevent duplication to work. Run the below command, verify the extents are tagged correctly using the following KQL    command:
```
.show table UsagePreliminary extents
| project Tags
``````

You should see the ADF Pipeline **RunId**, followed by the **month** (ex: 20230801-20230831)

![img](/images/manual_deployment26.png)

5.  Test the deduplication logic by rerunning the export job and making sure that the extents get replaced by new extents with a new RunID    but the same date range.

## Always Other Options

In this article we utilized ADF to manage the ingestion to a Kusto Database and orchestrate elimination of duplicates. But that's not the only way to handle this workflow.

-   An Azure Function could be used instead of ADF with [ADX Bindings for Azure Functions](https://techcommunity.microsoft.com/t5/azure-data-explorer-blog/azure-data-explorer-kusto-bindings-for-azure-functions-public/ba-p/3828472). However, that would be a code-first aproach, instead of using the low-code UI in ADF.
-   A [KQL materialize view](https://learn.microsoft.com/en-us/azure/data-explorer/kusto/management/materialized-views/materialized-view-overview) could be used instead of [.drop extents](https://learn.microsoft.com/azure/data-explorer/kusto/management/drop-extents) command to eliminate duplicates. As long as you understand which columns make the records unique, add the tag as values in an additional column or build a *surrogate-key* to uniquely identify or filter by, then a KQL materialize view would work. You wouldn't need to delete data either. Consider that records may be updated throughout the month (ie. prices change, etc) the materialized view would need to utilize an [arg_max()](https://learn.microsoft.com/azure/data-explorer/kusto/query/max-aggfunction) instead of a [take_any()](https://learn.microsoft.com/azure/data-explorer/kusto/query/take-any-aggfunction) for the deduplication.

### Summary

After implementing this example, the cost information for your Azure subscription(s) will be refreshed daily into your ADX database - as of the previous day.

Having this data in ADX can unlock other scenarios such as:
-   Join this information with other metadata to enrich and provide new
    insights.

-   Running ad-hoc queries in seconds to look/dashboard over the
    historical spend.

-   Build powerful visuals in your favorite tool such as ADX Dashboards,
    Power BI, Grafana, Tableau, Jupyter Notebooks, etc...