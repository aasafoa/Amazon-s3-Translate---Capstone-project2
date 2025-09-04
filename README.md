# Amazon-s3-Translate---Capstone-project2
Cloudlingo
# AWS IaC Translation Project 🚀

## Overview
This project demonstrates how to **design, develop, and implement an Infrastructure-as-Code (IaC) solution on AWS** using **CloudFormation or Terraform**. It integrates **AWS Translate** for automated language translation and **Amazon S3** for object storage. A **Python script with Boto3** processes translation requests from JSON files and stores both input and translated output in S3 buckets.

The solution can be deployed as:
- A **standalone script** (local execution).
- An **event-driven Lambda function** (automation).

This project is beginner-friendly and designed to run **within AWS Free Tier** limits.

---

## 🏗️ Architecture
1. **Request Bucket** → Upload JSON files containing text + metadata (source & target languages).
2. **Lambda (or Python Script)** → Reads input file, calls AWS Translate, generates output.
3. **Response Bucket** → Stores translated JSON file.
4. **IAM Role/Policy** → Grants permissions for S3 + Translate + Lambda.


---

## 📌 Objectives
- Provision AWS infrastructure with **CloudFormation/Terraform**.
- Enable **automated translation** via AWS Translate.
- Store results securely in Amazon S3.
- Keep costs at **zero (Free Tier)**.

---

## 🔑 Prerequisites
- AWS Account (Free Tier eligible).
- AWS CLI installed and configured (`aws configure`).
- Python 3.x installed with Boto3 (`pip install boto3`).
- Basic knowledge of JSON.

---

## ⚙️ Step 1: Create S3 Buckets + IAM Role with IaC

### Terraform Example (`main.tf`)
```hcl
provider "aws" {
  region = "us-east-1"
}

# Input bucket
resource "aws_s3_bucket" "request_bucket" {
  bucket = "translate-request-bucket-demo"
  force_destroy = true
}

# Output bucket
resource "aws_s3_bucket" "response_bucket" {
  bucket = "translate-response-bucket-demo"
  force_destroy = true
}

# IAM Role
resource "aws_iam_role" "lambda_role" {
  name = "lambda-translate-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "lambda.amazonaws.com"
      }
    }]
  })
}

# Attach Policies
resource "aws_iam_role_policy_attachment" "translate_policy" {
  role       = aws_iam_role.lambda_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonTranslateFullAccess"
}

resource "aws_iam_role_policy_attachment" "s3_policy" {
  role       = aws_iam_role.lambda_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonS3FullAccess"
}
---

## ⚙️ Step 2: Python Script with Boto3

We’ll create a Python script that:
1. Reads a JSON file from the **request S3 bucket**.
2. Submits the text to **AWS Translate**.
3. Saves the translated text to the **response S3 bucket**.

### Code: `translate_script.py`
```python
import boto3
import json
import sys

# Initialize clients
translate = boto3.client("translate")
s3 = boto3.client("s3")

def translate_text(bucket, key, target_lang="fr"):
    # Download JSON file
    obj = s3.get_object(Bucket=bucket, Key=key)
    data = json.loads(obj["Body"].read())

    source_text = data.get("text", "")
    source_lang = data.get("source_language", "en")
    target_lang = data.get("target_language", target_lang)

    # Translate text
    result = translate.translate_text(
        Text=source_text,
        SourceLanguageCode=source_lang,
        TargetLanguageCode=target_lang
    )

    translated_text = result["TranslatedText"]

    # Save output JSON
    output_data = {
        "original_text": source_text,
        "translated_text": translated_text,
        "source_language": source_lang,
        "target_language": target_lang
    }

    output_key = key.replace(".json", "_translated.json")
    s3.put_object(
        Bucket="translate-response-bucket-demo",
        Key=output_key,
        Body=json.dumps(output_data)
    )

    print(f"✅ Translation complete. File saved to response bucket as {output_key}")

# Run locally
if __name__ == "__main__":
    bucket = sys.argv[1]
    key = sys.argv[2]
    translate_text(bucket, key)
