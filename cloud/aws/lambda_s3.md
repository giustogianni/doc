# Lambda 
## Scenario
In the current practical example, we create a Lambda function which simply convert `.csv` files uploaded in an S3 bucket to `json` format.

Resources
* S3 bucket - `bucket-name`
* Lambda function 
* Role associated to Lambda with associated read \& write permissions on specific bucket
* EventBridge Rule

## Lambda
### General
Lambda definition requires
* Function name, *e.g.* `lambda-json-converter`
* Runtime, *e.g.* Python 3.10
* Architecture, *e.g.* x86_64
* Permissions, by default create a new execution role `lambda-json-converter-xxxx` with basic Lambda permissions and permissions to upload logs to Amazon CloudWatch Logs

### Code
```python
import pandas as pd

def lambda_handler(event, context):
    try:
        # Get the S3 bucket and key from the event
        
        # for EventBridge triggered events
        s3_bucket = event['detail']['bucket']['name']
        s3_key = event['detail']['object']['key']
        
        # for S3 triggered events
        #s3_bucket = event['Records'][0]['s3']['bucket']['name']
        #s3_key = event['Records'][0]['s3']['object']['key']
        
        # Check if the file uploaded is a CSV file
        if not s3_key.endswith('.csv'):
            raise ValueError(f"The uploaded file {s3_key} is not a CSV file.")

        # Get the file name and location
        file_name = s3_key.split('/')[-1]
        file_prefix = file_name.split('.')[0]

        # Read the CSV file 
        df = pd.read_csv(f"s3://{s3_bucket}/{s3_key}", delimiter=',')

        # Convert to JSON 
        df.to_json(f"s3://{s3_bucket}/{file_prefix}.json")

        return {'statusCode': 200, 'body': f"Converted {file_name} to JSON successfully."}

    except Exception as e:
        print(f"Error converting file: {e}, {event}")
        return {'statusCode': 500, 'body': f"Error converting file: {str(e)}"}
    
```

### Layers
Step 1 - Create a virtual environment
```console
python3 -m venv aws-fsspec
```

Step 2 - Activate newly created environment
```console
source aws-fsspec/bin/activate
```

Step 3 - Install dependency
```console
pip install fsspec
```

Step 4 - Create new `python` folder and copy dependencies in it
```console
cp -r /home/giusto/.virtualenvs/aws-fsspec/lib/python3.10/site-packages/ .
```

Result
```
.
└── python
    ├── _distutils_hack
    ├── distutils-precedence.pth
    ├── fsspec
    ├── fsspec-2024.2.0.dist-info
    ├── pip
    ├── pip-22.0.2.dist-info
    ├── pkg_resources
    ├── setuptools
    └── setuptools-59.6.0.dist-info
```

Step 5 - Zip dependencies
```console
zip -r fsspec.zip python
```

Step 6 - Create layer in AWS
* `Lambda` > `Layers`
* `Create layer`
    * Name: `AWSSDK<package>-Python<version>`, *e.g.* AWSSDKfsspec-Python310
    * Upload `.zip` 
    * Select architecture, *e.g.* x86_64
    * Select compatible runtime, *e.g.* Python 3.10

Step 7 - Add newly created layer to Lambda function

Step 8 - Repeat steps 1 to 7 to add more layers, current scenario requires 
* Pandas - already provided by AWS
* fsspec - create layer
* s3fs - create layer

## Role \& Policy
With a Lambda function comes a role for which we can define specific Permissions policies. 

Step 1 - Select Role

Step 2 - `Permissions` tab

Step 3 - `Add permissions` > `Create inline policy`

Step 4 - Define policy in Policy Editor
* Service: `S3`
* Actions allowed:
    * `GetObject` - Grants permission to retrieve objects from Amazon S3
    * `PutObject` - Grants permission to add an object to a bucket
* Resources
    * Specific - add ARN `arn:aws:s3:::bucket-name/*`

Result
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject"
            ],
            "Resource": "arn:aws:s3:::bucket-name/*"
        }
    ]
}
```

## Trigger upon S3 upload
Triggers from EventBridge or S3 are possible.

### S3 Trigger
#### Add Trigger
Step 1 - Select Lambda function > `Monitor` tab > `Add trigger`

Step 2 - Trigger config
* Source: `S3`
* Bucket: `s3/bucket-name`
* Event types: `PUT`
* Suffix: `.csv`

#### Event
```json
{
    "Records": [
        {
            "eventVersion": "2.1", 
            "eventSource": "aws:s3", 
            "awsRegion": "us-east-1", 
            "eventTime": "2024-03-07T13:17:15.508Z", 
            "eventName": "ObjectCreated:Put", 
            "userIdentity": {
                "principalId": "AWS:EXAMPLE:emailaddress@email.com"
            }, 
            "requestParameters": {
                "sourceIPAddress": "1.2.3.4"
            }, 
            "responseElements": {
                "x-amz-request-id": "EXAMPLE123456789", 
                "x-amz-id-2": "EXAMPLE123/5678abcdefghijklambdaisawesome/mnopqrstuvwxyzABCDEFGH"
            }, 
            "s3": {
                "s3SchemaVersion": "1.0", 
                "configurationId": "testConfigRuleId", 
                "bucket": {
                    "name": "bucket-name", 
                    "ownerIdentity": {
                        "principalId": "EXAMPLE"
                    }, 
                    "arn": "arn:aws:s3:::bucket-name"
                }, 
                "object": {
                    "key": "filename.csv", 
                    "size": 348393, 
                    "eTag": "0123456789abcdef0123456789abcdef", 
                    "sequencer": "0A1B2C3D4E5F678901"
                }
            }
        }
    ]
}
```

### EventBridge Trigger
#### Rule
Step 0 - Select name

Step 1 - Define Event pattern
* Service: `S3`
* Event type: `Amazon S3 Event Notification`. **WARNING** in S3, enable bucket to send notifications
    * From S3: `bucket-name` > `Properties` > `Amazon EventBridge` > `Edit` > `On`
* Event type 1: Specific event(s) > `Object Created`
* Event type 2: Specific bucket(s) > `bucket-name`
* `Edit pattern` to manually set `suffix` matching to avoid *recursive invocation* (if same bucket) 
    * Requirement: pattern structure needs to match event json, *i.e.* `detail` > `object` > `key`.

Result
```json
{
  "source": ["aws.s3"],
  "detail-type": ["Object Created"],
  "detail": {
    "bucket": {
      "name": ["bucket-name"]
    },
    "object": {
      "key": [{
        "suffix": ".csv"
      }]
    }
  }
}
```

Step 2 - Define Targets
* Target: `Lambda functions`
* Function: select already defined function

#### Event
```json
{
    "version": "0", 
    "id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx", 
    "detail-type": "Object Created", 
    "source": "aws.s3", 
    "account": "123456789012", 
    "time": "2024-03-07T13:17:15Z", 
    "region": "us-east-1", 
    "resources": ["arn:aws:s3:::bucket-name"], 
    "detail": {
        "version": "0", 
        "bucket": {
            "name": "bucket-name"
        }, 
        "object": {
            "key": "filename.csv", 
            "size": 348393, 
            "etag": "0123456789abcdef0123456789abcdef", 
            "sequencer": "0A1B2C3D4E5F678901"
        }, 
        "request-id": "N4N7GDK58NMKJ12R", 
        "requester": "123456789012", 
        "source-ip-address": "1.2.3.4", 
        "reason": "PutObject"
    }
}
```