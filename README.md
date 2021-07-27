# Reddit sentiment analysis using Amazon Comprehend

## Overview
The purpose of this project is to analyse user sentiments on Reddit feeds. Each reddit post would be classified to an overall sentiment (Positive, Negative, Neutral, Mixed) with a score given to each. Entities such as Location, Quantity, Date will be identified as well.

## Architecture
![sentiment architecture](https://user-images.githubusercontent.com/76123658/127139261-0bf9ecf4-08a5-42d8-bae6-5355b1cfb6e5.png)
When triggered by the CloudWatch scheduled events, Lambda function will update the corresponding .csv file of the subreddit with the new posts. The S3 bucket will contain all the .csv files and Amazon Athena will perform SQL queries on these files. Amazon Comprehend is used to perform sentiment analysis on the text.

## Option 1: CloudFormation

## Option 2: Manual Deployment

### 2.1 Create a S3 bucket
This S3 bucket is used to store the raw data from reddit as .csv format, each subreddit is stored as a separate object. Each .csv file will be created if not present, and new posts will be updated to the file from the last saved post.

### 2.2 Create a Lambda function with runtime dependencies: 

#### 2.2.1 Create the function
Create a new **IAM Role** and specify the following permissions to allow the Lambda function to write/read to S3 and write to CloudWatch Logs as the policy. Edit *your-bucket-name* and *your-account-id* accordingly.
```
{
  "Version": "2012-10-17",
  "Statement": [{
  "Effect": "Allow",
  "Action": [
  "s3:GetObject",
  "s3:PutObject",
  "s3:PutObjectAcl",
  "s3:GetObjectAcl"
  ],
  "Resource": "arn:aws:s3:::<your-bucket-name>/*"
  },{
  "Effect": "Allow",
  "Action": [
  "logs:CreateLogStream",
  "logs:PutLogEvents"
  ],
  "Resource": "arn:aws:logs:us-east-1:<your-account-id>:*"
  },{
  "Effect": "Allow",
  "Action": "logs:CreateLogGroup",
  "Resource": "*"
  }]
}
```
Create a new **Lambda function** and specify the runtime as Python 3.8. <br/>

Select *‘Use an existing role’* for the execution role and search for the newly created role, then create.<br/>

Copy and paste the code from lambda_function_github.py to the lambda_function.py on the console (Modified from [achernyshova](https://github.com/achernyshova/Reddit-NLP-Classification)) and edit the bucket name  accordingly. Edit the subreddits to be analysed under the TOPICS list.<br/>

Deploy updated lambda_function.py.<br/>

#### 2.2.2 Add dependencies under Layers
Since the lambda_function.py uses ‘requests’ library which is not in the standard library. To run the script, search for your region and the ‘requests’ package to find the appropriate ARN [here](https://github.com/keithrozario/Klayers/tree/master/deployments/python3.8/arns]). <br/>

Select *‘Add a layer’* under Layers and select *‘Specify an ARN’*. Input the ARN.<br/>

Run a test to check if objects are created in the S3 bucket.<br/>

### 2.3 Schedule CloudWatch events for Lambda function

Access your Lambda function on the Lambda console, select *‘Add trigger’*. <br/>

Create a new rule and under *‘Schedule expression’*, specify the frequency you want the lambda function to be invoked.<br/>
Eg. rate(15 minutes) to invoke every 15 minutes. <br/>

![lambda cloudwatch](https://user-images.githubusercontent.com/76123658/127140010-31241430-e19b-4d5c-b27d-d2c57e187ffa.png)


### 2.4 Setting up Athena
Within Athena console, specify a S3 bucket for query result location under ‘Settings’ <br/>

Create a **new table**, with 4 columns (id, subreddit, title, description).<br/>
```
CREATE EXTERNAL TABLE IF NOT EXISTS <your-database-name>.<your-table-name> (
  `id` string,
  `subreddit` string,
  `title` string,
  `description` string 
)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe'
WITH SERDEPROPERTIES (
  'serialization.format' = ',',
  'field.delim' = '|'
) LOCATION 's3://<your-bucket-name>/'
TBLPROPERTIES ('has_encrypted_data'='false');
```
<img src="https://user-images.githubusercontent.com/76123658/127139919-d270cd5d-c1ec-47f2-8b2d-bd2079aaecfd.png" width="215" height="364">

Test if Athena is able to query from the S3 bucket.<br/>
```
SELECT * FROM "<your-database-name>"." <your-table-name>" limit 10;
```

### 2.5 Setting up Amazon Comprehend

#### 2.5.1 Install the text analytics UDF
Install the **TextAnalyticsUDFHandler** application [here](https://console.aws.amazon.com/lambda/home?region=us-east-1#/create/app?applicationId=arn:aws:serverlessrepo:us-east-1:912625584728:applications/TextAnalyticsUDFHandler)

#### 2.5.2 Sentiment & Entities analysis table (JSON format) 
Create a new table with sentiments and entities columns added at the end of the original table. <br/>
```
CREATE TABLE <your-analysis-table-name> WITH (format='parquet') AS
USING
   EXTERNAL FUNCTION detect_sentiment_all(col1 VARCHAR, lang VARCHAR) RETURNS VARCHAR LAMBDA 'textanalytics-udf',
   EXTERNAL FUNCTION detect_entities_all(col1 VARCHAR, lang VARCHAR) RETURNS VARCHAR LAMBDA 'textanalytics-udf'
SELECT *, 
   detect_sentiment_all(description, 'en') AS sentiment,
   detect_entities_all(description, 'en') AS entities
FROM <your-table-name>
```

Test if the sentiments and entities columns are created. <br/>
```
SELECT * FROM "<your-database-name>"."<your-analysis-table-name>" limit 10;
```
![sentiment](https://user-images.githubusercontent.com/76123658/127139765-28b4b5a0-0adc-4329-824e-b007711642ef.png)


#### 2.5.3 Sentiment analysis table (Column format)
Create a new table to tabulate each sentiment score as an individual column. <br/>
```
CREATE TABLE <your-final-analysis-table-name>  WITH (format='parquet') AS
SELECT 
   id, subreddit, title, 
   CAST(JSON_EXTRACT(sentiment,'$.sentiment') AS VARCHAR) AS sentiment,
   CAST(JSON_EXTRACT(sentiment,'$.sentimentScore.positive') AS DOUBLE ) AS positive_score,
   CAST(JSON_EXTRACT(sentiment,'$.sentimentScore.negative') AS DOUBLE ) AS negative_score,
   CAST(JSON_EXTRACT(sentiment,'$.sentimentScore.neutral') AS DOUBLE ) AS neutral_score,
   CAST(JSON_EXTRACT(sentiment,'$.sentimentScore.mixed') AS DOUBLE ) AS mixed_score,
   description
FROM <your-analysis-table-name>
```
![sentiment final](https://user-images.githubusercontent.com/76123658/127139693-713b2af7-12df-41c9-aa22-3af9e3361331.png)


#### 2.5.4 Entities analysis table (Column format)
Create a table consisting of the entities found in the description and its associated category and score. <br/>
```
CREATE TABLE <your-entities-table-name> WITH (format='parquet') AS
SELECT 
   id, subreddit, title,  
   CAST(JSON_EXTRACT(entity_element, '$.text') AS VARCHAR ) AS entity,
   CAST(JSON_EXTRACT(entity_element, '$.type') AS VARCHAR ) AS category,
   CAST(JSON_EXTRACT(entity_element, '$.score') AS DOUBLE ) AS score,
   CAST(JSON_EXTRACT(entity_element, '$.beginOffset') AS INTEGER ) AS beginoffset,
   CAST(JSON_EXTRACT(entity_element, '$.endOffset') AS INTEGER ) AS endoffset,
   description
FROM
(
   SELECT * 
   FROM
      (
      SELECT *,
      CAST(JSON_PARSE(entities) AS ARRAY(json)) AS entities_array
      FROM reddit_data_with_text_analysis
      )
   CROSS JOIN UNNEST(entities_array) AS t(entity_element)
)
```
![entities](https://user-images.githubusercontent.com/76123658/127139520-424a5256-7880-4bc8-b590-85a4c8eb4a15.png)

#### 2.5.5 Further sentiment analysis
Aggregate functions like maximum, minimum, average etc. can be generated from Athena queries as well. <br/>
```
SELECT AVG(positive_score) AS average_positive_score,
   MAX(positive_score) AS max_positive_score,
   MIN(positive_score) AS min_positive_score,
   AVG(negative_score) AS average_negative_score,
   MAX(negative_score) AS max_negative_score,
   MIN(negative_score) AS min_negative_score,
   AVG(neutral_score) AS average_neutral_score,
   MAX(neutral_score) AS max_neutral_score,
   MIN(neutral_score) AS min_neutral_score,
   AVG(mixed_score) AS average_mixed_score,
   MAX(mixed_score) AS max_mixed_score,
   MIN(mixed_score) AS min_mixed_score

FROM <your-final-analysis-table-name>
```
![further sentiment](https://user-images.githubusercontent.com/76123658/127139413-2e86e3cc-c582-45a4-a02f-b722ee66ce2c.png)

[Amazon QuickSight](https://aws.amazon.com/quicksight/), a business intelligence (BI) service, can also be used to generate a dashboard and gain further insights on the data.

## FAQ
**1)	Unable to run the Lambda function and received the following error:**
```
{
  "errorMessage": "Unable to import module 'lambda_function': No module named 'requests'",
  "errorType": "Runtime.ImportModuleError",
  "stackTrace": []
}
```
Solution: Need to add the ‘requests’ runtime dependency to Layers. Refer to 2.2.2.

**2)	Athena query results show the data for each cell disorganised.** <br/> 

Solution: Refer to 2.4 and ensure the delimiter is set for ‘|’ and not ‘,’. This is to ensure that the commas in the description do not split the text to the next column.



