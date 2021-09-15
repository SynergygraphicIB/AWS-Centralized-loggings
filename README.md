# AWS-Centralized-loggings
How to Centralize Loggings into one account in AWS within one organization
Development  XXX0495 = 222222222222
Security XXX2340 = 111111111111

### PreFlight Check
1. Intermedial to advance level in Python. So, you can adapt and customized the `lambda.zip` files to your need an use cases.
2. Basic to intermedial level in json to edit the json file policy.json to modify it if needed to your use case, since we give granular limited access to AWS resources.
3. One AWS Account known as the **"Logging"** to centralize the logs from the **"Source-acccount"**.
4. A second AWS Account in which the CloudWatch Log Group can be used as the source for the logs data an has us-east-1 in **"Logging"** as Log Destination Endpoint
5. The AWS Command Line Interface (AWS CLI) properly installed, configured, and updated. Version 2 ‐ is the most recent major version of the AWS CLI and supports all of the latest features.

## List of AWS Resources used in the Auto Replicate Parameter Entries workflow
1. IAM
2. Lambda
3. CloudWatch
4. CloudTrail
5. SSM Parameter Store
6. The AWS Command Line Interface (AWS CLI)
7. ElasticSearch* (Need to add this step yet)

## List of Programming languages used
1. Python 3.9

## Required IAM Roles and Policies
In this case `Identity and Access Management (IAM)` is a global element, so do not worry about what region it is configured in you profile in the CLI. However, though some AWS Services like `EventBridge`, `CloudWatch`, and `Lambda` are regional. Therefore, be sure you are have us-east-1 (N. Virginia) as region for most of the purposes of this project. 
We need 2 roles; **assume-role-for-logging** and **centralized-logs**  with limited granular permissions to interact with other AWS Services such as: Lambda, IAM, SSM, and CloudWatch Logs for this project

The following limited policy will be attached to the role **centralized-logs**

**central-logging-policy.json** - IAM Policy to authorize *centralized-logs* to create a subscription filter in CloudWatch, get Parameter, and assume role
See `policy.json`

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "logs:DeleteSubscriptionFilter",
                "logs:DescribeSubscriptionFilters",
                "logs:PutSubscriptionFilter",
                "ssm:GetParameters",
                "ssm:GetParameter"
            ],
            "Resource": "*"
        },
        {
            "Sid": "VisualEditor1",
            "Effect": "Allow",
            "Action": "sts:AssumeRole",
            "Resource": "arn:aws:iam::596294512340:role/assume-role-for-logging"
        }
    ]
}
```
The following limited policy will be attached to the role **assume-role-for-logging**. 
To provide write permissions to CloudWatch Logs.
See the AWS Managed Policy `AWSLambdaBasicExecutionRole`

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": "*"
        }
    ]
}
```

### Amazon EventBridge
**EventRule.json** - This rule filters *CreateLogGroup* and  *CreateLogStream* events coming from `AWS API Call via CloudTrail` and uses as a target *centralized-logs*.
Take a pick at the rule here...

```json
	{
  "source": [
    "aws.logs"
  ],
  "detail-type": [
    "AWS API Call via CloudTrail"
  ],
  "detail": {
    "eventSource": [
      "logs.amazonaws.com"
    ],
    "eventName": [
      "CreateLogGroup",
      "CreateLogStream"
    ]
  }
}
```

Note: Sometimes when creating or editing complex custom rules such as when using prefix feature it is necessary to create them in EventBridge or to update the very same rules. If it is done CloudWatch directly it may not not work. Hence, we best configure the rules in EventBridge even though the end result is also shown in CloudWatch. 

# Paso 1.- Crear centralized-logs Role en 222222222222

## 1.1 

```
export AWS_PROFILE=development        
aws iam create-role --role-name centralized-logs --assume-role-policy-document '{"Version": "2012-10-17","Statement": [{ "Effect": "Allow", "Principal": {"Service": "lambda.amazonaws.com"}, "Action": "sts:AssumeRole"}]}'
```

**Output**

```json
{
    "Role": {
        "Path": "/",
        "RoleName": "centralized-logs",
        "RoleId": "AROAV2E2RDEX5MW4GU6KE",
        "Arn": "arn:aws:iam::222222222222:role/centralized-logs",
        "CreateDate": "2021-09-13T14:38:42+00:00",
        "AssumeRolePolicyDocument": {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Effect": "Allow",
                    "Principal": {
                        "Service": "lambda.amazonaws.com"
                    },
                    "Action": "sts:AssumeRole"
                }
            ]
        }
    }
}
```

## 1.2 Ahora pegarle la politica 

Asegurarse que se esta en la carpeta del proyecto… /logging-project

Y el archivo json se llama policy.json
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "logs:DeleteSubscriptionFilter",
                "logs:DescribeSubscriptionFilters",
                "logs:PutSubscriptionFilter",
                "ssm:GetParameters",
                "ssm:GetParameter"
            ],
            "Resource": "*"
        },
        {
            "Sid": "VisualEditor1",
            "Effect": "Allow",
            "Action": "sts:AssumeRole",
            "Resource": "arn:aws:iam::111111111111:role/replicate_log_assume_role"
        }
    ]
}
```


Ahora ejecutamos desde el CLI el comando 

```
aws iam create-policy --policy-name central-logging-policy --policy-document file://policy.json
```

**Output**
```json
{
    "Policy": {
        "PolicyName": "central-logging-policy",
        "PolicyId": "ANPAV2E2RDEXRDVWAACTJ",
        "Arn": "arn:aws:iam::222222222222:policy/central-logging-policy",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 0,
        "PermissionsBoundaryUsageCount": 0,
        "IsAttachable": true,
        "CreateDate": "2021-09-13T15:07:58+00:00",
        "UpdateDate": "2021-09-13T15:07:58+00:00"
    }
}
```

## 1.3 To attach the policy to the newly created role

```
aws iam attach-role-policy --role-name centralized-logs --policy-arn arn:aws:iam::222222222222:policy/central-logging-policy
```

Now lets go to Security Account ***

# Paso 2.- Crear Role assume-role-for-logging en 111111111111
```
Export AWS_PROFILE=security
```

Se puede chequear cual profile estoy usando… 
```
aws configure list
```

## 2.1
Comando CLI para crear role
```
aws iam create-role --role-name assume-role-for-logging --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"AWS":"arn:aws:iam::222222222222:role/centralized-logs"},"Action":"sts:AssumeRole"}]}'
```
**Output**
```json
{
    "Role": {
        "Path": "/",
        "RoleName": "assume-role-for-logging",
        "RoleId": "AROAYVVPMF3KIJP4Q24BT",
        "Arn": "arn:aws:iam::111111111111:role/assume-role-for-logging",
        "CreateDate": "2021-09-13T14:28:24+00:00",
        "AssumeRolePolicyDocument": {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Effect": "Allow",
                    "Principal": {
                        "AWS": "arn:aws:iam::222222222222:role/centralized-logs"
                    },
                    "Action": "sts:AssumeRole"
                }
            ]
        }
    }
}
```

## 2.2

Comando CLI para pegar policy
```
aws iam attach-role-policy --role-name assume-role-for-logging --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

there is not output

# Paso 3 To créate a “centralized-logs” lambda function 

## 3.1 Create lambda function
```
aws lambda create-function \
    --function-name centralized-logs \
    --zip-file fileb://lambda.zip \
    --role arn:aws:iam::222222222222:role/centralized-logs\
    --handler lambda_function.lambda_handler \
    --runtime python3.9
```
**Output:**
```json
{
    "FunctionName": "centralized-logs",
    "FunctionArn": "arn:aws:lambda:us-east-1:222222222222:function:centralized-logs",
    "Runtime": "python3.9",
    "Role": "arn:aws:iam::222222222222:role/centralized-logs",
    "Handler": "lambda.handler",
    "CodeSize": 1651,
    "Description": "",
    "Timeout": 3,
    "MemorySize": 128,
    "LastModified": "2021-09-13T16:28:36.134+0000",
    "CodeSha256": "HgW7rkaOikKYVZeQrOE9Yi+bDYrhx1oYp9y5b0zszTs=",
    "Version": "$LATEST",
    "TracingConfig": {
        "Mode": "PassThrough"
    },
    "RevisionId": "53edc92e-a99c-4fca-94b2-4dee0794670a",
    "State": "Active",
    "LastUpdateStatus": "Successful",
    "PackageType": "Zip"
}
```
## 3.2 Grant CloudWatch Logs the permission to execute your function. Use the following command, replacing the placeholder account with your own account and the placeholder log group with the log group to process:
```
aws lambda add-permission \
    --function-name "centralized-logs" \
    --statement-id "centralizesLogs" \
    --principal "logs.amazonaws.com" \
    --action "lambda:InvokeFunction" \
    --source-arn "arn:aws:logs:us-east-1:222222222222:log-group:*" \
    --source-account "222222222222"	
```

OUTPUT:
```json
{
    "Statement": "{\"Sid\":\"centralizesLogs\",\"Effect\":\"Allow\",\"Principal\":{\"Service\":\"logs.amazonaws.com\"},\"Action\":\"lambda:InvokeFunction\",\"Resource\":\"arn:aws:lambda:us-east-1:222222222222:function:centralized-logs\",\"Condition\":{\"StringEquals\":{\"AWS:SourceAccount\":\"222222222222\"},\"ArnLike\":{\"AWS:SourceArn\":\"arn:aws:logs:us-east-1:222222222222:log-group:*\"}}}"
}
```

# Paso 4. – Crear una regla en CloudWatch

Aquí vamos a crear una regla para ejecutar la lambda con una serie de comandos por pasos 

## 4.1 Crear la regla “sent-logs-to-lambda”

```
aws events put-rule --name "sent-logs-to-lambda" --event-pattern "{\"source\":[\"aws.logs\"],\"detail-type\":[\"AWS API Call via CloudTrail\"],\"detail\":{\"eventSource\":[\"logs.amazonaws.com\"],\"eventName\":[\"CreateLogGroup\",\"CreateLogStream\"]}}"
```
**OUTPUT:**
```json
{
    "RuleArn": "arn:aws:events:us-east-1:222222222222:rule/sent-logs-to-lambda"
}
```

## 4.2 To grant permissions to the lambda “centralized-logs” to allow it to be invoked by "arn:aws:events:us-east-1:222222222222:rule/sent-logs-to-lambda"

```
aws lambda add-permission \
--function-name centralized-logs \
--statement-id my-send-logs-to-lambda-permission \
--action 'lambda:InvokeFunction' \
--principal events.amazonaws.com \
--source-arn arn:aws:events:us-east-1:222222222222:rule/sent-logs-to-lambda
```

**OUTPUT:**

```json
{
    "Statement": "{\"Sid\":\"my-send-logs-to-lambda-permission\",\"Effect\":\"Allow\",\"Principal\":{\"Service\":\"events.amazonaws.com\"},\"Action\":\"lambda:InvokeFunction\",\"Resource\":\"arn:aws:lambda:us-east-1:222222222222:function:centralized-logs\",\"Condition\":{\"ArnLike\":{\"AWS:SourceArn\":\"arn:aws:events:us-east-1:222222222222:rule/sent-logs-to-lambda\"}}}"
}

```

## 4.3 To add the target to the rule arn arn:aws:events:us-east-1:222222222222:rule/sent-logs-to-lambda to the lambda function centralized-logs

```
aws events put-targets --rule sent-logs-to-lambda --targets file://targets.json
```
OUTPUT:

```json
{
    "FailedEntryCount": 0,
    "FailedEntries": []
}
```

# Paso 5.- Crear un entry en el Parameter Store

## 5.1 Follow the commands to create and entry at the Parameter Store for the Log Destination
```
aws ssm put-parameter \
    --name "LogDestination" \
    --type "String" \
    --value "arn:aws:lambda:us-east-1:222222222222:function:centralized-logs" \
    --overwrite	
```

**OUTPUT:**
```json
{
    "Version": 1,
    "Tier": "Standard"
}
```

## 5.2 Follow the commands to create and entry at the Parameter Store to assume the role in Security

```
aws ssm put-parameter \
    --name "LogArnAssume" \
    --type "String" \
    --value "arn:aws:iam::111111111111:role/assume-role-for-logging" \
    --overwrite
    
 ```

**OUTPUT:**
```json
{
    "Version": 1,
    "Tier": "Standard"
}

```
**** Testearlo ******
```
aws logs put-log-events --log-group-name testreplicatelogs --log-stream-name teststream1 --log-events "[{\"timestamp\":1631554348, \"message\": \"Simple logs replication Test\"}]"
```
