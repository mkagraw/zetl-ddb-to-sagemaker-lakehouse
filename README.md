# Amazon DynamoDB zero-ETL integration with Amazon SageMaker Lakehouse and data visualization using Sagemaker Unified Studio

[Amazon DynamoDB](https://aws.amazon.com/dynamodb/) [zero-ETL integration](https://aws.amazon.com/what-is/zero-etl/) with [Amazon SageMaker Lakehouse](https://aws.amazon.com/sagemaker/) allows you to run analytics workloads on your DynamoDB data without having to set up and manage extract, transform, and load (ETL) pipelines. You can now seamlessly consolidate your data from different tables and databases into SageMaker Lakehouse, giving you the ability to run holistic analytics across all your data. 

The solution outlines a step-by-step approach to:

- Integrate DynamoDB with SageMaker Lakehouse, enabling a seamless data flow.
- Use SageMaker Unified Studio with its generative SQL assistant powered by [Amazon Q](https://aws.amazon.com/q/), making it effortless to explore, analyze, and query operational and analytical data.

## Step-by-step

1. Open an [AWS CloudShell](uhttps://aws.amazon.com/cloudshell/) terminal window & clone the repo:

    ```git clone https://github.com/mkagraw/zetl-ddb-to-sagemaker-lakehouse.git```

2. cd to the repo folder **zetl-ddb-to-sagemaker-lakehouse**, replace ```<ACCOUNT-ID>``` in the files with your own AWS account ID using the following command:

    ```cd zetl-ddb-to-sagemaker-lakehouse; find . -type f -exec sed -i 's/"<ACCOUNT-ID>"/XXXXXXXXXXXX/g' {} \\;```

**Note:** Below commands creates all AWS resources in the **us-east-1** AWS Region. If you want to change your Region, DynamoDB table name, S3 URI location, AWS Glue database, or AWS Glue table name, update the files accordingly.

3. Create the DynamoDB table CustomerAccounts and verify table creation status:

    ```aws dynamodb create-table --cli-input-json file://CustomerAccountsTable.json --region us-east-1```

    ```aws dynamodb describe-table --table-name CustomerAccounts --region us-east-1```

4. Enable point-in-time recovery (PITR) for the DynamoDB table **CustomerAccounts**

    ```aws dynamodb update-continuous-backups --table-name CustomerAccounts --point-in-time-recovery-specification PointInTimeRecoveryEnabled=true --region us-east-1```

5. Load sample data into the DynamoDB table CustomerAccounts:

```aws dynamodb batch-write-item --request-items file://sample-data-load-1.json --region us-east-1```

6. To access data from your source DynamoDB table, AWS Glue requires access to describe the table, and export data from it. DynamoDB recently introduced a feature that allows configuring a [resource-based access control policy](https://docs.aws.amazon.com/glue/latest/dg/zero-etl-sources.html#zero-etl-config-source-dynamodb). Add and verify the following resource policy for the DynamoDB table CustomerAccounts, enabling the zero-ETL integration to access DynamoDB table data.

**NOTE:** Before running the following commands, replace ```<ACCOUNT-ID>``` with your own AWS account ID

    aws dynamodb put-resource-policy --resource-arn arn:aws:dynamodb:us-east-1:"<ACCOUNT-ID>":table/CustomerAccounts --policy file://CustomerAccounts_ResourcePolicy.json --region us-east-1

    aws dynamodb get-resource-policy --resource-arn arn:aws:dynamodb:us-east-1:"<ACCOUNT-ID>":table/CustomerAccounts --region us-east-1

7. Create an S3 bucket, folder, and an [AWS Glue database](https://docs.aws.amazon.com/glue/latest/dg/zero-etl-prerequisites.html#zero-etl-setup-target-resources-glue-database) by providing the S3 URI location:

    ```aws s3 mb s3://glue-zetl-target-&lt;ACCOUNT-ID&gt;-us-east-1 --region us-east-1```

    ```aws s3api put-object --bucket glue-zetl-target-&lt;ACCOUNT-ID&gt;-us-east-1 --key customerdb/``` 

aws glue create-database --database-input '{  
    "Name": "customerdb",  
    "Description": "Glue database for storing metadata with S3 location",  
    "LocationUri": "s3://glue-zetl-target-"<ACCOUNT-ID>"-us-east-1/customerdb/",  
    "Parameters": {  
        "CreatedBy": "AWS CLI"  
    }  
}' --region us-east-1

1. For integrations that use an AWS Glue database, add the permissions to the [catalog resource-based access policy](https://docs.aws.amazon.com/glue/latest/dg/zero-etl-prerequisites.html#zero-etl-setup-target-resources) to allow for integrations between source and target. You can access the catalog settings under the Data Catalog on the AWS Glue console, or run the following command:

aws glue put-resource-policy --policy-in-json file://glue_catalog_rbac_policy.json

1. Create an IAM policy for source resources. This will enable the zero-ETL integration to access your connection.

aws iam create-policy \\  
\--policy-name zETLSPolicy \\  
\--policy-document file://source_policy.json

1. Create an IAM policy for target resources:

aws iam create-policy \\  
    --policy-name zETLTPolicy \\  
    --policy-document file://target_policy.json

1. Create an IAM policy to enable [Amazon CloudWatch](http://aws.amazon.com/cloudwatch) logging:

aws iam create-policy \\  
    --policy-name zETLCWPolicy \\  
    --policy-document file:\\\\cloud_watch_policy.json

1. Create an IAM role with the following trust permissions and attach the IAM policies to the target role:

aws iam create-role --role-name zETLTRole --assume-role-policy-document '{  
    "Version": "2012-10-17",  
    "Statement": \[  
      {  
        "Effect": "Allow",  
        "Principal": {  
          "Service": "glue.amazonaws.com"  
        },  
        "Action": "sts:AssumeRole"  
      }  
    \]  
  }'

aws iam attach-role-policy \\  
    --role-name zETLTRole \\  
    --policy-arn arn:aws:iam::&lt;ACCOUNT-ID&gt;:policy/zETLSPolicy  

aws iam attach-role-policy \\  
    --role-name zETLTRole \\  
    --policy-arn arn:aws:iam::&lt;ACCOUNT-ID&gt;:policy/zETLTPolicy  

aws iam attach-role-policy \\  
    --role-name zETLTRole \\  
    --policy-arn arn:aws:iam::&lt;ACCOUNT-ID&gt;:policy/zETLCWPolicy

## Create the zero-ETL integration

With the prerequisites complete, complete the following steps to create the zero-ETL integration:

1. On the DynamoDB console, choose **Integrations** in the navigation pane.
2. Choose **Create integration**, then choose **Amazon SageMaker Lakehouse**. You will be redirected to Glue console.

!

1. For your data source, select **Amazon DynamoDB**, then choose **Next**.

![]
Next, you need to configure the source and target details.

1. In the **Source details** section, for **DynamoDB table**, choose the table CustomerAccounts.
2. In the **Target details** section, specify the Data Catalog name, target database name (customerdb), and target IAM role you created previously (zETLTargetRole).

![]
Now you have options to configure the output.

1. For **Data partitioning**, you can either use DynamoDB table keys for partitioning, or specify custom partition keys.
2. Choose **Next**.

![]
1. Under **Configure integration**, you can configure your data encryption. You can use [AWS Key Management Service](https://aws.amazon.com/kms/) (AWS KMS), or a custom encryption key.
2. Enter a name for the integration, and choose **Next**.

![]
1. Review the configurations, and choose **Create and launch integration**.

![]
After the initial data ingestion is complete, the zero-ETL integration will be ready for use. The completion time varies depending on the size of your source DynamoDB table.

![]
On the AWS Glue console, choose **Tables** under **Data Catalog** in the navigation pane, and open your table to observe more details, including the schema. The zero-ETL integration uses Iceberg to transform related data formats and structure in your DynamoDB data into appropriate formats in Amazon S3.

![]
Lastly, you can confirm that your data is available in your S3 bucket.

![]
## Conclusion

In this post, we walked through the prerequisites and steps required to set up a zero-ETL integration from DynamoDB to SageMaker Lakehouse. In [Part 2,](file:///Users/mkagraw/Downloads/part2_blog_link) we show how to set up SageMaker Lakehouse using SageMaker Unified Studio, and demonstrate how you can run analytics using the capabilities of SageMaker Lakehouse.

To learn more about zero-ETL integration, refer to [DynamoDB zero-ETL integration with Amazon SageMaker Lakehouse](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/amazon-sagemaker-lakehouse-for-DynamoDB-zero-etl.html).

# Amazon DynamoDB zero-ETL integration with Amazon SageMaker Lakehouse – Part 2

In [Part 1](file:///Users/mkagraw/Downloads/part1_blog_link) of this series, we walked through the prerequisites and steps to create a [zero-ETL integration](https://aws.amazon.com/what-is/zero-etl/) between [Amazon DynamoDB](https://aws.amazon.com/dynamodb/) and [Amazon SageMaker Lakehouse](https://aws.amazon.com/sagemaker/lakehouse/). In this post, we continue setting up SageMaker Lakehouse using [Amazon SageMaker Unified Studio](https://aws.amazon.com/sagemaker/unified-studio/), and show sample analytics that a financial services team can run to understand their customer’s behavior and spend.

## Set up SageMaker Unified Studio

SageMaker Unified Studio is an integrated development environment (IDE) for data, analytics, and AI. You can discover your data and put it to work using familiar AWS tools to complete end-to-end development workflows in a single governed environment. Such tools may include data analysis, data processing, model training, generative AI application building, and more. You can create or join projects to collaborate with your teams, share AI and analytics artifacts securely, and discover and use your data stored in [Amazon Simple Storage Service](http://aws.amazon.com/s3) (Amazon S3), [Amazon Redshift](http://aws.amazon.com/redshift), and other data sources through SageMaker Lakehouse. As AI and analytics use cases converge, you can transform how data teams work together with SageMaker Unified Studio.

Follow the steps in [An integrated experience for all your data and AI with Amazon SageMaker Unified Studio](https://aws.amazon.com/blogs/big-data/an-integrated-experience-for-all-your-data-and-ai-with-amazon-sagemaker-unified-studio/) to create a SageMaker Unified Studio domain and project. For this example, we name the project CustomerDB_Project. On the project overview page, copy the project role Amazon Resource Name (ARN) for future use.

![]
## Import AWS Glue Data Catalog assets to the SageMaker Unified Studio project

Follow the instructions and script provided in the [GitHub repo](https://docs.aws.amazon.com/sagemaker-unified-studio/latest/userguide/bring-resources-scripts.html) to bring in existing resources to the SageMaker Unified Studio project CustomerDB_Project.

1. Retrieve details about the [AWS Identity and Access Management](https://aws.amazon.com/iam/) (IAM) user or role whose credentials will be used in running the script. In the following commands, the assumed role Admin is used:

aws sts get-caller-identity

1. Add the executor (IAM user or role, in our case the Admin role) as the [AWS Lake Formation](https://aws.amazon.com/lake-formation/%5d) data lake administrator in the AWS Region where you will execute the script:

aws lakeformation put-data-lake-settings \\  
\--data-lake-settings '{"DataLakeAdmins":\[{"DataLakePrincipalIdentifier":"arn:aws:iam::&lt;ACCOUNT-ID&gt;:role/Admin"}\]}' \\  
\--region us-east-1

1. Download the bring_your_own_gdc_assets.py script from [GitHub](https://docs.aws.amazon.com/sagemaker-unified-studio/latest/userguide/bring-resources-scripts.html), and execute the following command:

python3 bring_your_own_gdc_assets.py \\  
—project-role-arn &lt;project-role-ARN&gt; \\  
—table-name customeraccounts \\  
—database-name customerdb \\  
—region us-east-1

At this point, the [AWS Glue](https://aws.amazon.com/glue) database customerdb and table CustomerAccounts should be visible in the project CustomerDB_Project under AwsDataCatalog.

![]
## Explore your data through SageMaker Lakehouse

In this section, we use the SageMaker Lakehouse data explorer to explore the imported table with [Amazon Athena](http://aws.amazon.com/athena). Complete the following steps:

1. On the project page, choose **Data**.
2. Under **Lakehouse**, expand AwsDataCatalog.
3. Expand your database customerdb.
4. Choose the customeraccounts table, then choose **Query with Athena**.
5. Choose **Run all**.

The following screenshot shows an example of the query result.

![]
You can also open a generative SQL assistant powered by [Amazon Q](https://aws.amazon.com/q/) to help your query authoring experience. For example, you can ask “calculate the total balance as a percentage of the credit limit outstanding per state and zipcode” in the Amazon Q assistant, and the query is automatically suggested.

You can choose **Add to querybook** to copy the suggested query to your querybook, and run it.

![]
Next, regarding the query generated by Amazon Q, let’s try a quick visualization to analyze the data distribution.

1. Choose the chart view icon.
2. Under **Structure**, choose **Traces**.
3. For **Type**, choose **Pie**.
4. For **Values**, choose balance_percentage.
5. For **Labels**, choose state.

The query result will display as a pie chart like the following example. You can customize the graph title, axis title, subplot styles, and more in the UI. The generated images can also be downloaded as PNG or JPEG files. After you’ve finished querying the data, you can choose to view the queries in your query history, and save them to share with other project members.

![]
For more information about reviewing query history, see [Review query history](https://docs.aws.amazon.com/sagemaker-unified-studio/latest/userguide/query-history.html). For more information about other operations you can do with the query editor, such as using generative AI to create SQL queries, see [SQL analytics](https://docs.aws.amazon.com/sagemaker-unified-studio/latest/userguide/sql-query.html).

Let’s ask Amazon Q to generate a query by providing the prompt “calculate the count of account status Past Due by state, zipcode.” The following screenshot shows the results.

![]
Next, we enter the query “pivot table where rows represent states, and columns represent account statuses with the corresponding counts.” The following screenshot shows the results.

![]
## Schema evolution

The zero-ETL integration uses [Apache Iceberg](https://iceberg.apache.org/) to transform related data formats and structure in your DynamoDB data into appropriate formats in Amazon S3. Iceberg provides a robust solution for managing table schemas in data lakes, enabling schema evolution (adding or removing columns) without disrupting data, or requiring costly rewrites. It uses metadata files to track schema changes, allowing for querying historical data even after schema modifications. For more information about zero-ETL integration general limitations, refer to [Limitations](https://docs.aws.amazon.com/glue/latest/dg/zero-etl-limitations.html).

To see how the zero-ETL integration handles schema evolution, let’s make some changes in the DynamoDB table CustomerAccounts. We will do the following:

1. Insert new items with matching attributes of an existing item with a few additional new attributes.
2. Update an item by adding new attributes.
3. Delete an item.

Open an [AWS CloudShell](https://aws.amazon.com/cloudshell/) terminal window and run the following commands:

aws dynamodb **put-item** --table-name CustomerAccounts \\

\--item '{

"CustomerID": {"S": "CUST0026"},

"AccountNumber": {"S": "ACC839001"},

"State": {"S": "WA"},

"ZipCode": {"S": "98101"},

"CreditLimit": {"N": "10140"},

"CurrentBalance": {"N": "8135.47"},

"LastPaymentDate": {"S": "2024-12-17"},

"AccountStatus": {"S": "Active"},

"InterestRate": {"N": "3.15"},

"email": { "S": "<sarah.wilson@example.com>" },

"custname": { "S": "Sarah Wilson" },

"username": { "S": "swilson789" },

"phone": { "S": "555-012-3456" },

"address": { "S": "789 Oak St, Chicago, IL 60601" },

"custcreatedt": { "S": "2023-04-01T09:00:00Z" },

"custupddt": { "S": "2023-04-01T09:00:00Z" }

}'  
<br/>aws dynamodb **update-item** --table-name CustomerAccounts \\

\--key '{ "CustomerID": { "S": "CUST0001" }, "AccountNumber": { "S": "ACC326093" } }' \\

\--update-expression "SET email = :val1, custname = :val2, username = :val3, phone = :val4, address = :val5, custcreatedt = :val6, custupddt = :val7" \\

\--expression-attribute-values '{

":val1": { "S": "<michael.taylor@example.com>" },

":val2": { "S": "Michael Taylor" },

":val3": { "S": "mtaylor123" },

":val4": { "S": "555-246-8024" },

":val5": { "S": "246 Maple Ave, Los Angeles, CA 90001" },

":val6": { "S": "2022-11-01T08:00:00Z" },

":val7": { "S": "2022-11-01T08:00:00" }

}'

aws dynamodb **delete-item** \\

\--table-name CustomerAccounts \\

\--key '{"CustomerID": {"S": "CUST0002"}, "AccountNumber": {"S": "ACC578303"}}'

The zero-ETL integration will take up to 15–30 minutes to propagate the change in the AWS Glue Data Catalog database.

Run the following query to validate the changed data:

select \* from awsdatacatalog.customerdb.customeraccounts where customerid in ('CUST0026', 'CUST0001', 'CUST00002');

## Time travel and version travel queries for historical analysis

Each Iceberg table maintains a versioned manifest of the Amazon S3 objects that it contains. Previous versions of the manifest can be used for time travel and version travel queries. _Time travel queries_ in Athena query Amazon S3 for historical data from a consistent snapshot as of a specified date and time. _Version travel queries_ in Athena query Amazon S3 for historical data as of a specified snapshot ID. For more information about time travel and version travel queries, see [Perform time travel and version travel queries](https://docs.aws.amazon.com/athena/latest/ug/querying-iceberg-time-travel-and-version-travel-queries.html).

Retrieve the snapshot ID using the following query:

select \* from "awsdatacatalog"."customerdb"."customeraccounts**$snapshots**";

Use the following version travel query for historical analysis:

select \* from awsdatacatalog.customerdb.customeraccounts **FOR VERSION AS OF** &lt;SNAPSHOT_ID&gt; where customerid in ('CUST0026', 'CUST0001', 'CUST00002');

Use the following time travel query for historical analysis:

SELECT \* FROM "awsdatacatalog"."customerdb"."customeraccounts" **FOR TIMESTAMP AS OF** (current_timestamp - interval '1' day)

## Monitor the zero-ETL integration

There are several options to obtain metrics on the performance and status of the DynamoDB zero-ETL integration with SageMaker Lakehouse.

On the AWS Glue console, choose **Zero-ETL integrations** in the navigation pane. You can choose your zero-ETL integration, and drill down to view Amazon CloudWatch logs and Amazon CloudWatch metrics to the integration.

![]
## Viewing Amazon CloudWatch logs for an integration

Zero-ETL integrations generate Amazon CloudWatch logs for visibility into your data movement. Log events are emitted to a default log group created in customer account. Log events may include successful ingestion events, failures experienced due to problematic data records at source, and data write errors due to schema changes or insufficient permissions.

![]
## Viewing Amazon CloudWatch metrics for an integration

Zero-ETL integration provides real-time operational insights through CloudWatch metrics, enabling proactive monitoring of data integration processes without direct querying of target Iceberg tables. When enabled by adding appropriate permissions on source and target processing roles, CloudWatch metrics are automatically emitted to the AWS/Glue/ZeroETL namespace after completion of each table ingestion operation. You can setup alarms on your CloudWatch metrics to get notified when a particular Ingestion Job fails.

![]
To learn about zero ETL monitoring, refer to [zero ETL monitoring](https://docs.aws.amazon.com/glue/latest/dg/zero-etl-monitoring.html)

## Pricing

AWS doesn’t charge an additional fee for the zero-ETL integration. You pay for existing DynamoDB and AWS Glue resources used to create and process the change data created as part of a zero-ETL integration. These include DynamoDB point-in-time recovery (PITR), DynamoDB exports for the initial and ongoing data changes to your DynamoDB data, additional AWS Glue storage for storing replicated data, and SageMaker compute on the target. For pricing on DynamoDB PITR and DynamoDB exports, see [Amazon DynamoDB pricing](https://aws.amazon.com/dynamodb/pricing/).

## Clean up

Complete the following steps to clean up your resources. When you delete a zero-ETL integration, your data isn’t deleted from the DynamoDB table or AWS Glue, but data changes happening after that point of time aren’t sent to SageMaker Lakehouse.

1. Delete the zero-ETL integration:
    1. On the AWS Glue console, choose **Zero-ETL integrations** in the navigation pane.
    2. Select the zero-ETL integration that you want to delete, and on the **Actions** menu, choose **Delete**.
    3. To confirm the deletion, enter confirm, and choose **Delete**.
2. Delete the SageMaker Unified Studio project, domain and VPC:
    1. On the SageMaker Unified Studio console, select your project, and on the **Action** menu, choose **Delete project**.
    2. On the SageMaker console, select the domain you want to delete, and choose **Delete**.
    3. On CloudFormation Stacks, select SageMakerUnifiedStudio-VPC, and choose **Delete**.
3. Delete the AWS Glue database:
    1. On the AWS Glue console, choose **Data Catalog** in the navigation pane.
    2. Select your database, and choose **Delete** to delete the database and its tables.
4. On the Amazon S3 console, delete the S3 folder and bucket you created.
5. On the IAM console, delete the IAM policies and role you created.

## Conclusion

In this post, we explained how you can set up a zero-ETL integration from DynamoDB to SageMaker Lakehouse to derive holistic insights across many applications, break data silos in your organization, and gain significant cost savings and operational efficiencies.

To learn more about zero-ETL integration, refer to [DynamoDB zero-ETL integration with Amazon SageMaker Lakehouse](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/amazon-sagemaker-lakehouse-for-DynamoDB-zero-etl.html).
