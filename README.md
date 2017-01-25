# A Quick and Easy End-to-End Data Solution
---
[comment]: <> (Do I use the term Turn-Key or Quick-and-Dirty or Out-of-the-box or Plug-and-Play or some other options?)

We recently had a professional sports team come to us as a potential client. As with any professional sports team, they had a variety of data sources and a need to perform analytics on that data. Their data sources were both internal and external, and in a variety of formats. The primary sources of their data included their CRM running in Microsoft Dynamics, Ticketmaster daily extracts, Point-of-Sale transaction data, and a variety of miscellaneous internal application data. They were wanting to bring all these data sources together to perform analytics to determine the key drivers of season ticket sales, just to start out with. They also wanted a turn-key solution that they would not need to babysit themselves or hire people with tech skills to handle. They needed to have dashboards for executives that would be able to point to a centralized data repository. The dashboards would need to be able to show high level summaries of the data as well as provide drill-down capabilities. The final requirement was that this all needed to be done yesterday.

[comment]: <> (Add Somethings about Data Driven Decision Making?)

![Image](https://upload.wikimedia.org/wikipedia/commons/thumb/1/1d/Major_sports_by_state.svg/500px-Major_sports_by_state.svg.png)

While discovering what the client needed, we also discovered something about professional sports teams. They have very large name recognition, however they are actually more like a small businesses. With such a large name recognition one would think they are like a large enterprise with an accompanying large budget. However they actually have a small back office and IT staff that's just there to support the bare minimum of functionality. Most of their expenditures are for players, and expenses for other things like analytics and databases need to be kept to a minimum.

This all seems like a *very* tall order, and there are *very* few options that can provide a wholistic and turn-key solution for their needs. Especially in such a short amount of time. And especially not on a small budget. The client essentially was looking for a move-in-ready data environment, and they wanted to be handed the keys right away.

![Image](http://www.timberlinespirits.com/wp-content/uploads/2015/04/Turnkey.jpg)

[//]: <> (I need to rewrite and revise this paragraph)

Now, typically one can't have their cake and eat it too. Usually it's said you can have fast, quality, or cheap pick any two, because one can't have all three. However, with a winning combination of Amazon Web Services (AWS) providing the building blocks, and 1Strategy performing the building and gluing together of those building blocks, it is possible to achieve all three fast, quality, and cheap.

![Image](https://www.bookmarc.com.au/wp-content/uploads/2015/02/brickimage-3.jpg)

The AWS building blocks we choose to use were S3 and RDS, and then some custom scripting to bring the pieces together. S3 was used as a landing zone for all of their data sources, and RDS was used as place to house the data after some processing and formatting. We decided to use RDS Aurora as it is an AWS Managed Service so the client would not need to worry about database maintenance, upgrades, or servicing. Aurora also provides a MySQL interface which made it easy to use with Excel, Tableau, and any other analysis tools the client may want to use.

Below are the steps we took to build out the data environment.

## Build Steps

* [Step 1 - Create an Aurora Cluster](#step-1)
* [Step 2 - Configure Aurora Connection](#step-2)
* [Step 3 - Connect to the Aurora Cluster](#step-3)
* [Step 4 - Load Data from S3 to Aurora](#step-4)
* [Step 5 - Format Data to Required Data Types](#step-5)
* [Step 6 - Query the Data](#step-6)
* [Step 7 - Connect to Data from Excel](#step-7)
* [Step 8 - Connect to Data from Tableau and Visualize](#step-8)

### <a name="step-1"></a>Step 1 - Create an Aurora Cluster
The first step is to create an Aurora cluster. This was done through the AWS Console.
  - Select RDS from the Services Menu
  - Select the database engine, in this case we chose Aurora for the reasons described above.
  - Next are the database details.
    * For this demo we started out with the defaults for DB Instance Class and Multi-AZ Deployment.
    * For DB Instance Identifier, specify a name that is unique for all DB instances owned by your AWS account in the current region. DB instance identifier is case insensitive, but stored as all lower-case, as in "mydbinstance"
    * Then specify the Master Username and Password
  - Next are configuring the Advanced Settings.
    * Leave most of the Advanced Settings with the default values with the exception of the Database Name value. For that specify a string of up to 64 alpha-numeric characters that define the name given to a database that Amazon RDS creates when it creates the DB instance, as in “mydb”.
  - Then click Launch DB Instance, and the Aurora database will be created.

### <a name="step-2"></a>Step 2 - Configure Aurora Connection
The next step is to configure a Role and Security Group for connecting to the Aurora Cluster, and then to associate the Role and Security Group to the Aurora Cluster.
  - Select EC2 from the Services Menu, and then select Security Groups from the sidebar.
  - Select Create Security Group, and then add an inbound rule for MYSQL/Aurora (Type: MYSQL/Aurora, Protocol: TCP, Port: 3306, Source: IP Address), and finally click create.
  - Next select IAM from the Services Menu, and then select Roles from the sidebar.
  - Select Create New Role.
    * For Role Name provide an appropriate name, like "my-aurora-role".
    * For Role Type select AWS Service Roles.
    * From the list select Amazon RDS (Allows RDS to call AWS services on your behalf.)
    * For Attach Policy just skip for now and click Next Step.
    * Then Review and click Create Role.
  - Next go back to Roles on the sidebar, then search for the role you just created, and then select it. (This Role is needed in order to move data from S3 to Aurora as part of the ETL process later on.)
    * On the Permissions tab select Attach Policy
    * Search for S3 and then select the "AmazonS3FullAccess" Policy.
    * Search for RDS and then select the "AmazonRDSFullAccess" Policy.
    * Select Attach Policy.
  - Next select RDS from the Services Menu, and the select Instances from the sidebar.
  - Select the instance used for the Cluster Endpoint. It should be the one with the Cluster Instance name *without* the AZ as part of the name.
    * Select Instance Actions, and from the drop down select Modify.
    * Under the Network & Security section and in the Security Group box select the Security Group previously created.
    * Scroll to the bottom and check the box for "Apply Immediately".
    * Then select Continue and then Modify DB Instance.
  - Next select Clusters from the sidebar, and select the cluster previously created.
    * Select Manage IAM Roles.
    * Select the Role previously created for Aurora from the drop down.
    * Select Done.

### <a name="step-3"></a>Step 3 - Connect to the Aurora Cluster
In this step you will connect to the Aurora Cluster using a JDBC driver and the following connection string pattern: `jdbc:mysql://<your-aurora-cluster-endpoint>.rds.amazonaws.com:3306/<your-db-name>`

For this example I use SQL Squirrel, but any application that uses JDBC will be able to connect in a similar way.

### <a name="step-4"></a>Step 4 - Load Data from S3 to Aurora
For this step I used SQL Squirrel to submit the code below to the Aurora Cluster to load sample data that was previously extracted from source and saved in S3.

It is also necessary to set up permissions for Aurora to access S3 before running this see: [link to AWS Documentation](http://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Aurora.Authorizing.AWSServices.html?shortFooter=true)

The code example below uses two different methods of loading data from S3. The first method loads all files from an S3 path, and the second one loads data from a single file on S3.

```SQL
CREATE TABLE demo_db.names (
  name_id INT(10) AUTO_INCREMENT,PRIMARY KEY (name_id),
  name VARCHAR(255),
  gender CHAR(1),
  usage_ct INT,
  load_dt DATETIME
);

LOAD DATA FROM S3 PREFIX 's3://test-bucket-00/names-data/'
INTO TABLE demo_db.names
      FIELDS TERMINATED BY ','
      LINES TERMINATED BY '\n'
      (name, gender, usage_ct)
SET load_dt = CURRENT_TIMESTAMP
;

CREATE TABLE demo_db.sales (
  sales_id INT(10) AUTO_INCREMENT,PRIMARY KEY (sales_id),
  Transaction_date VARCHAR(255),
  Product VARCHAR(255),
  Price VARCHAR(255),
  Payment_Type VARCHAR(255),
  Name VARCHAR(255),
  City VARCHAR(255),
  State VARCHAR(255),
  Country VARCHAR(255),
  Account_Created VARCHAR(255),
  Last_Login VARCHAR(255),
  Latitude VARCHAR(255),
  Longitude VARCHAR(255),
  load_dt DATETIME
);

LOAD DATA FROM S3 FILE 's3://test-bucket-00/SampleSalesJan2009.csv'
INTO TABLE demo_db.sales
      FIELDS TERMINATED BY ','
      LINES TERMINATED BY '\n'
      (Transaction_date,Product,Price,Payment_Type,Name,City,State,Country,Account_Created,Last_Login,Latitude,Longitude)
SET load_dt = CURRENT_TIMESTAMP
;

CREATE TABLE demo_db.customers (
  cust_id INT(10) AUTO_INCREMENT,PRIMARY KEY (cust_id),
  first_name VARCHAR(255),
  last_name VARCHAR(255),
  company_name VARCHAR(255),
  address VARCHAR(255),
  city VARCHAR(255),
  county VARCHAR(255),
  state VARCHAR(255),
  zip VARCHAR(255),
  phone1 VARCHAR(255),
  phone2 VARCHAR(255),
  email VARCHAR(255),
  web VARCHAR(255),
  load_dt DATETIME
);

LOAD DATA FROM S3 FILE 's3://test-bucket-00/sample_customers.txt'
INTO TABLE demo_db.customers
      FIELDS TERMINATED BY ','
      LINES TERMINATED BY '\n'
      (first_name,last_name,company_name,address,city,county,state,zip,phone1,phone2,email,web)
SET load_dt = CURRENT_TIMESTAMP
;
```
### <a name="step-5"></a>Step 5 - Format Data to Required Data Types
When the sample data has been loaded in the previous step, it is all loaded as strings. This was done to ease the loading of data and to make it easier to do data typing in a way that works best with MySQL/Aurora.
```SQL
-- Convert strings to data types
CREATE TABLE sales_format AS SELECT
  sales_id,
  STR_TO_DATE(transaction_date, '%m/%d/%y %H:%i') AS tran_datetime,
  product,
  CAST(price AS DECIMAL) AS price_in_dollars,
  payment_type,
  name,
  city,
  state,
  country,
  STR_TO_DATE(account_created, '%m/%d/%y %H:%i') AS acct_created_datetime,
  STR_TO_DATE(last_login, '%m/%d/%y %H:%i') AS last_login_datetime,
  latitude*1 AS lat_num,
  longitude*1 AS lon_num,
  load_dt
FROM demo_db.sales
;
```

### <a name="step-6"></a>Step 6 - Query the Data
Querying data from Aurora.
```SQL
------------------------------------------------
-- Query names table
select * from demo_db.names limit 100
select count(*) from demo_db.names
select max(usage_ct) from names
select * from names where usage_ct >= 90000
select name,gender,sum(usage_ct) as sum_ct from names group by name,gender order by sum_ct desc
------------------------------------------------
-- Query sales table
select * from demo_db.sales limit 100;
select cast(transaction_date as DATETIME) as tran_datetime,transaction_date from demo_db.sales limit 100;
select payment_type, count(*) as ct from sales group by 1 order by 2 desc;
select price, count(*) as ct from sales group by 1 order by 2 desc;
select product, count(*) as ct from sales group by 1 order by 2 desc;
select product,price, count(*) as ct from sales group by 1,2 order by 3 desc;
select * from sales_format;
------------------------------------------------
-- Query customers table
select * from demo_db.customers limit 100;

```
### <a name="step-7"></a>Step 7 - Connect to Data from Excel
To connect to the Aurora database from Excel is a simple process of using Excel's built in query tools for connecting to external data sources using ODBC connections

![Image](https://blogs.office.com/wp-content/uploads/2015/08/Working-with-external-data-in-Excel-2016-for-Mac-1.png)

Once the connection to the Aurora database has been created then you can use the MSQuery to write SQL and query your data similar to the examples in the previous section. Once your query has been run, the results from the query will be in your Excel spreadsheet. From there you can use all the powers of Excel like Pivot Tables and Charts to visualize and interact with the data.

### <a name="step-8"></a>Step 8 - Connect to Data from Tableau and Visualize
This section describes how to connect Tableau to a MySQL or Aurora database and set up the data source.
Before you begin, gather this connection information:
  - Name of the server that hosts the database you want to connect to
  - User name and password
  - Are you connecting to an SSL server?
Make the connection and set up the data source

On the start page, under Connect, click MySQL, and then do the following:
    1. Enter the name of the server that hosts the database.
    2. Enter the user name and password, and then click Sign In.
    3. Select the Require SSL check box when connecting to an SSL server.
From the Database drop-down list, select a database or use the text box to search for a database by name.

Under Table, select a table or use the text box to search for a table by name.

Drag the table to the canvas, and then click the sheet tab to start your analysis.

Use custom SQL to connect to a specific query rather than the entire data source. For more information.

## Summary and Conclusions.
In the end a customer can get up and running and using an end-to-end data solution very quickly and easily using AWS's services, and a few short scripts they have an end-to-end solution.
