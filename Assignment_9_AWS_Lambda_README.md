# Assignment 9 -- AWS Lambda Practice

This README contains all 10 Lambda questions with minimal Python code,
dummy test inputs, and short setup steps.

## Prerequisites

-   An AWS account.
-   Create a DynamoDB table named `Students`.
-   Use **`PRN` as the partition key (String)** for the student
    exercises below.
-   Create Lambda functions with the Python 3.12 runtime.
-   Attach the required DynamoDB permissions to each Lambda execution
    role.
-   For S3 exercises, create an S3 bucket in the same AWS Region as the
    Lambda function.

> **Important:** These examples assume the `Students` table has `PRN` as
> its partition key. If your existing table uses `student_id`, change
> each `PRN` key in the code to `student_id` and use that field
> consistently.

------------------------------------------------------------------------

## 1. Add two numbers

**Question:** Create an AWS Lambda function that takes two numbers as
input using a test event and returns their addition.

**Lambda code:**

``` python
def lambda_handler(event, context):
    a = event['num1']
    b = event['num2']

    return {
        'statusCode': 200,
        'result': a + b
    }
```

**Dummy test event:**

``` json
{
    "num1": 10,
    "num2": 20
}
```

**Short steps:** 1. Create a Lambda function using Python. 2. Paste the
code and click **Deploy**. 3. Create a test event with the JSON above.
4. Click **Test**. The result should be `30`.

------------------------------------------------------------------------

## 2. Accept and print student details

**Question:** Create a Lambda function that accepts input values (name,
PRN, course) from a test event and prints them.

**Lambda code:**

``` python
def lambda_handler(event, context):
    print("Name:", event['name'])
    print("PRN:", event['PRN'])
    print("Course:", event['course'])

    return {
        'statusCode': 200,
        'body': 'Student details printed'
    }
```

**Dummy test event:**

``` json
{
    "name": "Sarfaraz Shaikh",
    "PRN": "123456",
    "course": "Computer Engineering"
}
```

**Short steps:** 1. Create a Python Lambda function. 2. Paste the code
and click **Deploy**. 3. Add the dummy JSON as a test event and click
**Test**. 4. View the printed details in **Monitor → View CloudWatch
logs**.

------------------------------------------------------------------------

## 3. Insert student details into DynamoDB

**Question:** Create a Lambda function that inserts student details into
a DynamoDB table.

**Lambda code:**

``` python
import json
import boto3

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Students')


def lambda_handler(event, context):
    table.put_item(
        Item={
            'PRN': event['PRN'],
            'name': event['name'],
            'course': event['course']
        }
    )

    return {
        'statusCode': 200,
        'body': json.dumps('Student inserted successfully')
    }
```

**Dummy test event:**

``` json
{
    "PRN": "123456",
    "name": "Sarfaraz Shaikh",
    "course": "Computer Engineering"
}
```

**Short steps:** 1. Create the `Students` DynamoDB table with partition
key `PRN` (String). 2. Create a Lambda function and attach permission
`dynamodb:PutItem` for this table. 3. Paste the code, deploy, and test
with the dummy event. 4. Open DynamoDB → `Students` → **Explore table
items** to verify the record.

------------------------------------------------------------------------

## 4. Read student details from DynamoDB using PRN

**Question:** Create a Lambda function that reads student details from
DynamoDB using PRN.

**Lambda code:**

``` python
import json
import boto3

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Students')


def lambda_handler(event, context):
    response = table.get_item(
        Key={'PRN': event['PRN']}
    )

    item = response.get('Item', {})

    return {
        'statusCode': 200 if item else 404,
        'body': json.dumps(item, default=str) if item else 'Student not found'
    }
```

**Dummy test event:**

``` json
{
    "PRN": "123456"
}
```

**Short steps:** 1. Create a Python Lambda function. 2. Attach
permission `dynamodb:GetItem` for the `Students` table. 3. Paste the
code, deploy, and test using an existing PRN. 4. The student record is
returned if found.

------------------------------------------------------------------------

## 5. Update a student's course name in DynamoDB

**Question:** Create a Lambda function that updates a student's course
name in DynamoDB.

**Lambda code:**

``` python
import json
import boto3

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Students')


def lambda_handler(event, context):
    response = table.update_item(
        Key={'PRN': event['PRN']},
        UpdateExpression='SET course = :c',
        ExpressionAttributeValues={
            ':c': event['course']
        },
        ConditionExpression='attribute_exists(PRN)',
        ReturnValues='UPDATED_NEW'
    )

    return {
        'statusCode': 200,
        'body': json.dumps(response['Attributes'])
    }
```

**Dummy test event:**

``` json
{
    "PRN": "123456",
    "course": "AWS Cloud Computing"
}
```

**Short steps:** 1. Create a Python Lambda function. 2. Attach
permission `dynamodb:UpdateItem` for the `Students` table. 3. Paste the
code, deploy, and test using an existing PRN. 4. Check the item in
DynamoDB to verify the updated course.

------------------------------------------------------------------------

## 6. Delete a student record from DynamoDB using PRN

**Question:** Create a Lambda function that deletes a student record
from DynamoDB using PRN.

**Lambda code:**

``` python
import json
import boto3

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Students')


def lambda_handler(event, context):
    response = table.delete_item(
        Key={'PRN': event['PRN']},
        ReturnValues='ALL_OLD'
    )

    return {
        'statusCode': 200,
        'body': json.dumps(response.get('Attributes', {}), default=str)
    }
```

**Dummy test event:**

``` json
{
    "PRN": "123456"
}
```

**Short steps:** 1. Create a Python Lambda function. 2. Attach
permission `dynamodb:DeleteItem` for the `Students` table. 3. Paste the
code, deploy, and test using an existing PRN. 4. Check the DynamoDB
table; the record should be deleted.

------------------------------------------------------------------------

## 7. Scan all student records from DynamoDB

**Question:** Create a Lambda function that scans all student records
from a DynamoDB table.

**Lambda code:**

``` python
import json
import boto3

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Students')


def lambda_handler(event, context):
    response = table.scan()
    students = response['Items']

    return {
        'statusCode': 200,
        'body': json.dumps(students, default=str)
    }
```

**Dummy test event:**

``` json
{}
```

**Short steps:** 1. Create a Python Lambda function. 2. Attach
permission `dynamodb:Scan` for the `Students` table. 3. Paste the code,
deploy, and test with an empty event. 4. The matching table records are
returned.

> **Note:** A single DynamoDB `scan()` call can return up to 1 MB. For a
> large table, use pagination with `LastEvaluatedKey` to retrieve every
> record.

------------------------------------------------------------------------

## 8. Check whether a PRN exists before inserting a student

**Question:** Create a Lambda function that checks whether a student PRN
already exists before inserting a new record.

**Lambda code:**

``` python
import json
import boto3

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Students')


def lambda_handler(event, context):
    prn = event['PRN']

    response = table.get_item(
        Key={'PRN': prn}
    )

    if 'Item' in response:
        return {
            'statusCode': 409,
            'body': json.dumps('Student PRN already exists')
        }

    table.put_item(
        Item={
            'PRN': prn,
            'name': event['name'],
            'course': event['course']
        }
    )

    return {
        'statusCode': 200,
        'body': json.dumps('Student inserted successfully')
    }
```

**Dummy test event:**

``` json
{
    "PRN": "123457",
    "name": "Rahul Patil",
    "course": "Information Technology"
}
```

**Short steps:** 1. Create a Python Lambda function. 2. Attach
permissions `dynamodb:GetItem` and `dynamodb:PutItem` for the `Students`
table. 3. Paste the code, deploy, and test with a PRN that is not
already present. 4. Test again with the same PRN; the function should
report that it already exists.

> **Note:** This simple check-then-insert example is suitable for
> practice. In production, use a conditional `put_item` with
> `attribute_not_exists(PRN)` to prevent duplicate inserts if two
> requests run at the same time.

------------------------------------------------------------------------

## 9. Trigger Lambda on S3 upload and print bucket name and file name

**Question:** Create a Lambda function that triggers when a file is
uploaded to an Amazon S3 bucket and prints the bucket name and file
name.

**Lambda code:**

``` python
def lambda_handler(event, context):
    for record in event['Records']:
        bucket = record['s3']['bucket']['name']
        file_name = record['s3']['object']['key']

        print("Bucket Name:", bucket)
        print("File Name:", file_name)

    return {
        'statusCode': 200,
        'body': 'S3 upload detected'
    }
```

**Dummy S3 event (for manual Lambda testing):**

``` json
{
    "Records": [
        {
            "s3": {
                "bucket": {
                    "name": "my-student-bucket"
                },
                "object": {
                    "key": "student.pdf"
                }
            }
        }
    ]
}
```

**Short steps:** 1. Create an S3 bucket. 2. Create and deploy the Lambda
function. 3. In Lambda, choose **Add trigger → S3**. 4. Select the
bucket and event type **All object create events**, then add the
trigger. 5. Upload a file to the bucket and check the Lambda CloudWatch
logs.

------------------------------------------------------------------------

## 10. Trigger Lambda on S3 upload and print uploaded file size

**Question:** Create a Lambda function that triggers on S3 upload and
prints the size of the uploaded file.

**Lambda code:**

``` python
def lambda_handler(event, context):
    for record in event['Records']:
        file_name = record['s3']['object']['key']
        file_size = record['s3']['object']['size']

        print("File Name:", file_name)
        print("File Size:", file_size, "bytes")

    return {
        'statusCode': 200,
        'body': 'File size retrieved successfully'
    }
```

**Dummy S3 event (for manual Lambda testing):**

``` json
{
    "Records": [
        {
            "s3": {
                "bucket": {
                    "name": "my-student-bucket"
                },
                "object": {
                    "key": "student.pdf",
                    "size": 204800
                }
            }
        }
    ]
}
```

**Short steps:** 1. Create an S3 bucket. 2. Create and deploy the Lambda
function. 3. Add an S3 trigger with **All object create events**. 4.
Upload a file to the bucket. 5. Check CloudWatch logs for the file name
and size in bytes.

------------------------------------------------------------------------

## Quick IAM permission reference

Attach only the permissions required by each function, scoped to the
`Students` table.

  -----------------------------------------------------------------------
  Exercise                            Required DynamoDB action
  ----------------------------------- -----------------------------------
  3\. Insert student                  `dynamodb:PutItem`

  4\. Read student                    `dynamodb:GetItem`

  5\. Update course                   `dynamodb:UpdateItem`

  6\. Delete student                  `dynamodb:DeleteItem`

  7\. Scan students                   `dynamodb:Scan`

  8\. Check and insert                `dynamodb:GetItem`,
                                      `dynamodb:PutItem`

  9--10. S3 event details             No S3 read permission needed just
                                      to read event metadata and print it
  -----------------------------------------------------------------------

For exercises 9 and 10, configure the S3 trigger on the Lambda function.
AWS will add or prompt for permission allowing S3 to invoke the
function.

## Useful reminders

-   Click **Deploy** after changing Lambda code.
-   Use the **Test** tab to run exercises 1--8.
-   S3 exercises 9--10 can be tested with the sample event, or by
    uploading a real file after configuring the trigger.
-   Find `print()` output in **Lambda → Monitor → View CloudWatch
    logs**.
-   Keep the Lambda function and S3 bucket in the same AWS Region for a
    straightforward S3 trigger setup.
