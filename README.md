```python
aws configure
```
The aws access key and the secrete key was created.
	
```bash
#!/bin/bash

set -x

#Store the AWS account ID in a variable

aws_account_id=$(aws sts get-caller-identity --query 'Account' --output text)

#Print the AWS account ID from the variable
echo "AWS Account ID: $aws_account_id"

#Set AWS region and bucket name

aws_region="us-east-1"
bucket_name="karthi-lambda"
lambda_func_name="s3-lamba-function"
role_name="s3-lambda-sns"
email_address="skarthi770@gmail.com"

#Create IAM Role for the project
role_response=$(aws iam create-role --role-name "s3-lambda-sns" --assume-role-policy-document '{
"Version": "2012-10-17",
"Statement": [{
    "Action": "sts:AssumeRole",
    "Effect": "Allow",
    "Principal": {
        "Service": [
        "lambda.amazonaws.com",
        "s3.amazonaws.com",
        "sns.amazonaws.com"
        ]
    }
}]
}')

#Extract the role ARN from the JSON response and store it in a variable
role_arn=$(echo "$role_response" | jq -r '.Role.Arn')

#Print the role ARN
echo "Role ARN: $role_arn"

#Attach Permissions to the Role

aws iam attach-role-policy --role-name $role_name --policy-arn arn:aws:iam::aws:policy/AWSLambda_FullAccess

aws iam attach-role-policy --role-name $role_name --policy-arn arn:aws:iam::aws:policy/AmazonSNSFullAccess

  

#Create the S3 bucket and capture the output in a variable
bucket_output=$(aws s3api create-bucket --bucket "$bucket_name" --region "$aws_region")

  

#Print the output from the variable
echo "Bucket creation output: $bucket_output"

  

#Upload a file to the bucket
aws s3 cp ./example_file.txt s3://"$bucket_name"/example_file.txt

#Create a zip file to upload Lambda Function
zip -r s3-lambda-function.zip ./s3-lambda-function

sleep 5

# Create a Lambda function

lambda_function_response=$(aws lambda create-function\
    --region "$aws_region"\
    --function-name $lambda_func_name\
    --runtime "python3.8"\
    --handler "s3-lambda-function/s3-lambda-function_handler"\
    --memory-size 128 \
    --timeout 30\
    --role "arn:aws:iam::774721023955:role/$role_name"\
    --zip-file "fileb://./s3-lambda-function.zip")

# Extract the Lambda function ARN from the response
LambdaFunctionArn=$(echo "$lambda_function_response" | jq -r '.FunctionArn')

# Print the Lambda function ARN
echo "Lambda Function ARN: $LambdaFunctionArn"

#Add Permissions to s3 bucket to invoke Lambda
aws lambda add-permission \
    --function-name "$lambda_func_name" \
    --statement-id "s3-lambda-sns"\
    --action "lambda:InvokeFunction" \
    --principal s3.amazonaws.com \
    --source-arn "arn:aws:s3:::$bucket_name"

#Create an S3 event trigger for the lambda function
aws s3api put-bucket-notification-configuration \
    --region "$aws_region" \
    --bucket "$bucket_name" \
    --notification-configuration '{
        "LambdaFunctionConfigurations": [{
                "LambdaFunctionArn": "'"$LambdaFunctionArn"'",
                "Events": ["s3:ObjectCreated:*"]
        }]
    }'

#Create an sns topic and save the topic ARN to a variable
topic_arn=$(aws sns create-topic --name s3-lambda-sns --output json | jq -r '.TopicArn')

#Add SNS publish permission to the Lambda Function
aws sns subscribe \
 --topic-arn "$topic_arn" \
 --protocol email \
 --notification-endpoint "$email_address"
# Publish SNS
aws sns publish \
    --topic-arn "$topic_arn" \
    --subject "A new object created in s3 bucket" \
    --message "You did it Karthi!!"
```

![[Pasted image 20240108124928.png]]

![[Pasted image 20240108125208.png]]

![[Pasted image 20240108125300.png]]
![[Pasted image 20240108125440.png]]
![[Pasted image 20240108125753.png]]
