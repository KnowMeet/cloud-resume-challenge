## Terraform

We have used an infrastructure as code (IaC) tool to automate the provisioning of the AWS resources required for the project. Terraform scripts were written to define the desired state of the infrastructure, making it easy to create, update, and version-control the AWS environment. Here, we have only created AWS Lambda as a code but we could create rest of the services as well using Terraform. 

For the complete code checkout **[IaC](/cloud-resume-challenge/IaC/)** folder. Follow below steps to create AWS Lambda using Terrafrom:

## 1. Providers 

The following code will inform Terraform to create resources on AWS. They are like connectors that allow Terraform to interact with specific cloud or service providers like AWS, Azure, or Google Cloud. Create providers.tf file and add the following code.

```tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# Configure the AWS Provider
provider "aws" {
  region = "us-east-1"
}

```

## 2.  main.tf

This file will have all the necessary code to build the resources such as AWS Lambda, etc. 

### 1. Lambda Function

This will create a simple lambda function.

```tf

resource "aws_lambda_function" "ResumeCountFunc" {
  # If the file is not in the current working directory you will need to include a
  # path.module in the filename.
  filename      = data.archive_file.zip.output_path
  source_code_hash = data.archive_file.zip.output_base64sha256
  function_name = "ResumeCountFunc"
  role          = aws_iam_role.iam_for_lambda.arn
  handler       = "ResumeCountFunc.lambda_handler"
  runtime = "python3.12"
  }
```

### 2. IAM Role

This will create an IAM role for the lambda function and attach cors functionality.

```tf

resource "aws_iam_role" "iam_for_lambda" {
  name               = "iam_for_lambda"
  assume_role_policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": "sts:AssumeRole",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Effect": "Allow",
      "Sid": ""
    }
  ]
}
EOF
}

```

### 3. Lambda code

To add a Lambda code, we need to create another folder [Lambda](/cloud-resume-challenge/IaC/lambda/ResumeCountFunc.py) and add the python code file. Apart from it, we will access it using *data* resource called *archive file*. This will generate zip file as an output for the Lambda function. 

```tf

data "archive_file" "zip" {
  type        = "zip"
  source_file = "${path.module}/lambda/ResumeCountFunc.py"
  output_path = "${path.module}/lambda/packaged.zip"
}

```

### 4. Fucntion URL

To generate Lambda Function URL, use the resource called *aws_lambda_function_url* as shown below.

```tf
resource "aws_lambda_function_url" "url" {
  function_name      = aws_lambda_function.ResumeCountFunc.function_name
  authorization_type = "NONE"
   cors {
    allow_credentials = true
    allow_origins     = ["*"]
    allow_methods     = ["*"]
    allow_headers     = ["date", "keep-alive"]
    expose_headers    = ["keep-alive", "date"]
    max_age           = 86400
  }
}

```

### 5. IAM Policy

The IAM role does not have the access to DynamoDB table, therefore we need to add IAM policy.

```tf
resource "aws_iam_policy" "iam_policy_cloud_resume" {

  name        = "aws_iam_policy_cloud__resume"
  path        = "/"
  description = "AWS IAM Policy for managing the resume project role"
    policy = jsonencode(
    {
      "Version" : "2012-10-17",
      "Statement" : [
        {
          "Action" : [
            "logs:CreateLogGroup",
            "logs:CreateLogStream",
            "logs:PutLogEvents"
          ],
          "Resource" : "arn:aws:logs:*:*:*",
          "Effect" : "Allow"
        },
        {
          "Effect" : "Allow",
          "Action" : [
            "dynamodb:BatchGetItem",
            "dynamodb:GetItem",
            "dynamodb:Query",
            "dynamodb:Scan",
            "dynamodb:BatchWriteItem",
            "dynamodb:PutItem",
            "dynamodb:UpdateItem"
          ],
          "Resource" : "arn:aws:dynamodb:*:*:table/view-counter" # Make sure to add your Dynamodb table name here
        },
      ]
  })
}

```

## 6. Attach role and Policy

It is imperative to attach both IAM role and Policy together so that we could retrieve view counts from the website.

```tf
resource "aws_iam_role_policy_attachment" "iam_dynamo" {
  role       = aws_iam_role.iam_for_lambda.name
  policy_arn = aws_iam_policy.iam_policy_cloud_resume.arn
}

```