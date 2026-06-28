# 07 — IAM, Lambda & DynamoDB

## IAM (Identity & Access Management)

### List users/roles/policies
```bash
aws iam list-users
aws iam list-roles
aws iam list-policies --scope AWS
aws iam list-policies --scope Local   # customer-managed
```

### Create user
```bash
aws iam create-user --user-name alice

# Create access keys for the user
aws iam create-access-key --user-name alice
```

### Attach policy to user
```bash
aws iam attach-user-policy \
  --user-name alice \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

### Create custom policy
```bash
aws iam create-policy \
  --policy-name MyBucketPolicy \
  --policy-document file://policy.json
```

Example `policy.json`:
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject"],
    "Resource": "arn:aws:s3:::my-bucket/*"
  }]
}
```

### Create role
```bash
# Trust policy file (who can assume this role)
# trust-policy.json → allows EC2 to assume the role
aws iam create-role \
  --role-name MyEC2Role \
  --assume-role-policy-document file://trust-policy.json

# Attach permissions to role
aws iam attach-role-policy \
  --role-name MyEC2Role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

### Instance profile (for EC2)
```bash
aws iam create-instance-profile --instance-profile-name MyProfile
aws iam add-role-to-instance-profile --instance-profile-name MyProfile --role-name MyEC2Role
```

### List policies attached to a user
```bash
aws iam list-attached-user-policies --user-name alice
```

### Simulate policy (test permissions)
```bash
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789012:user/alice \
  --action-names s3:ListBucket s3:GetObject
```

## Lambda

### List functions
```bash
aws lambda list-functions
```

### Invoke a function
```bash
# Synchronous (with payload)
aws lambda invoke --function-name my-function \
  --payload '{"key": "value"}' output.json

# Async (event-based)
aws lambda invoke --function-name my-function \
  --invocation-type Event \
  --payload file://event.json output.txt

# Get the result
cat output.json
```

### Create a function
```bash
# Package your code into a ZIP
zip function.zip index.js

# Create the function
aws lambda create-function \
  --function-name my-function \
  --runtime nodejs20.x \
  --role arn:aws:iam::123456789012:role/lambda-execution-role \
  --handler index.handler \
  --zip-file fileb://function.zip
```

### Update function code
```bash
aws lambda update-function-code \
  --function-name my-function \
  --zip-file fileb://function.zip
```

### List function URLs
```bash
aws lambda list-function-url-configs --function-name my-function
```

### Set environment variables
```bash
aws lambda update-function-configuration \
  --function-name my-function \
  --environment Variables={DB_HOST=mydb.com,DB_PORT=5432}
```

## DynamoDB

### List tables
```bash
aws dynamodb list-tables
```

### Create table
```bash
aws dynamodb create-table \
  --table-name Users \
  --attribute-definitions AttributeName=id,AttributeType=S \
  --key-schema AttributeName=id,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
```

### Put item
```bash
aws dynamodb put-item \
  --table-name Users \
  --item '{
    "id": {"S": "user1"},
    "name": {"S": "Alice"},
    "age": {"N": "30"},
    "active": {"BOOL": true}
  }'
```

### Get item
```bash
aws dynamodb get-item \
  --table-name Users \
  --key '{"id": {"S": "user1"}}'
```

### Query (by primary key)
```bash
aws dynamodb query \
  --table-name Users \
  --key-condition-expression "id = :id" \
  --expression-attribute-values '{":id": {"S": "user1"}}'
```

### Scan (full table)
```bash
aws dynamodb scan --table-name Users
```

### Update item
```bash
aws dynamodb update-item \
  --table-name Users \
  --key '{"id": {"S": "user1"}}' \
  --update-expression "SET age = :new_age" \
  --expression-attribute-values '{":new_age": {"N": "31"}}'
```

### Delete item
```bash
aws dynamodb delete-item \
  --table-name Users \
  --key '{"id": {"S": "user1"}}'
```

### Delete table
```bash
aws dynamodb delete-table --table-name Users
```

---

### GUI Equivalent
**IAM Console** → Users, Roles, Policies.  
**Lambda Console** → Functions, Create, Test.  
**DynamoDB Console** → Tables, Items, Query/Scan.

### Next: [08 — Productivity](./08-productivity.md)
