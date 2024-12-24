# opensearch_sagemaker_multilingual
Amazon OpenSearch using Amazon SageMaker for multilingual searching.

** this is repo is intended for demo purposes ONLY. For production workloads, make sure you follow security best practices such as creating resources within a VPCs, ensuring that IAM policies/roles have granular access control, etc. Resources should be deleted after done playing with various queries.

 **Prerequesites:**
 1. Must have AWS account

These first couple steps set up an Amazon SageMaker notebook to use for the OpenSearch SageMaker multilingual Demo. You can run these commands on AWS Cloudshell after logging into your AWS account.

Please replace the placeholders ```${AWS::Region}``` and ```${AWS::AccountId}``` with your actual AWS region and account ID.

### Step 1 - Create an S3 bucket to store models
```
S3_BUCKET_NAME="${AWS_REGION}-${AWS_ACCOUNT_ID}-opensearch-sagemaker-demo-models"
aws s3api create-bucket --bucket $S3_BUCKET_NAME --region $AWS_REGION
aws s3api put-public-access-block \
    --bucket $S3_BUCKET_NAME \
    --public-access-block-configuration "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"
```
```
aws s3api put-bucket-encryption \
    --bucket $S3_BUCKET_NAME \
    --server-side-encryption-configuration '{"Rules": [{"ApplyServerSideEncryptionByDefault": {"SSEAlgorithm": "aws:kms"}}]}'
```
```
aws s3api put-bucket-policy --bucket $S3_BUCKET_NAME --policy '{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Deny",
            "Principal": "*",
            "Action": "s3:*",
            "Resource": [
                "arn:aws:s3:::'"$S3_BUCKET_NAME"'",
                "arn:aws:s3:::'"$S3_BUCKET_NAME"'/*"
            ],
            "Condition": {
                "Bool": {"aws:SecureTransport": "false"}
            }
        }
    ]
}'

echo "Bucket '$S3_BUCKET_NAME' created and configured successfully."
```

### Step 2 - Create an IAM role for Sagemaker and attach policies

Frist we must clone this repo so we have access to the code.
``` 
git clone https://github.com/jtrollin/opensearch_sagemaker_multilingual.git
cd opensearch_sagemaker_multilingual/
```
Now we need to update the policies to have the right account id and region, please remember to update `${AWS::AccountId}` with your account id and `${AWS::Region}` with your region.
```
sed -i 's/__AccountId__/${AWS::AccountId}/g' sagemaker_policy.json
sed -i 's/__Region__/${AWS::Region}/g' sagemaker_policy.json

aws iam create-role \
    --role-name ${AWS::Region}-${AWS::AccountId}-SageMaker-Execution-demo-role \
    --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"sagemaker.amazonaws.com"},"Action":"sts:AssumeRole"}]}'
```
```
aws iam attach-role-policy \
    --role-name ${AWS::Region}-${AWS::AccountId}-SageMaker-Execution-demo-role \
    --policy-arn arn:aws:iam::aws:policy/AmazonSageMakerFullAccess

policy_arn=$(aws iam create-policy \
    --policy-name sagemaker-policy \
    --policy-document file://sagemaker_policy.json \
    --query 'Policy.Arn' \
    --output text)
    
aws iam attach-role-policy \
    --policy-arn $policy_arn \
    --role-name ${AWS::Region}-${AWS::AccountId}-SageMaker-Execution-demo-role
```
You can see the policy we added below or by opening the sagemaker_policy.json
```
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Action": [
				"s3:PutObject",
                "s3:GetObject",
                "s3:DeleteObject",
                "s3:ListBucket",
                "s3:CreateBucket",
                "s3:PutBucketPublicAccessBlock",
                "s3:PutEncryptionConfiguration",
                "s3:PutBucketPolicy"
			],
			"Resource": [
				"arn:aws:s3:::__Region__-__AccountId__-opensearch-sagemaker-demo-models/*",
				"arn:aws:s3:::__Region__-__AccountId__-opensearch-sagemaker-demo-models/"
			],
			"Effect": "Allow"
		},
		{
			"Action": [
				"es:CreateDomain",
				"es:DescribeDomain",
				"es:DeleteDomain",
				"es:ESHttpPost",
				"es:ESHttpPut",
				"es:CreateElasticsearchDomain",
				"es:DescribeDomainHealth"
			],
			"Resource": [
				"*",
				"arn:aws:es:__Region__:__AccountId__:domain/*"
			],
            "Effect": "Allow"
		}
	]
}
```

### Step 3 - Create an IAM role for Amazon Opensearch and create a custom policy
Within Cloudshell, run the following command:

```
sed -i 's/__AccountId__/${AWS::AccountId}/g' opensearch_policy.json
sed -i 's/__Region__/${AWS::Region}/g' opensearch_policy.json

aws iam create-role \
    --role-name ${AWS::Region}-${AWS::AccountId}-SageMaker-OpenSearch-demo-role \
    --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"opensearchservice.amazonaws.com"},"Action":"sts:AssumeRole"}]}'

policy_arn=$(aws iam create-policy \
    --policy-name opensearch-policy \
    --policy-document file://opensearch_policy.json \
    --query 'Policy.Arn' \
    --output text)
    
aws iam attach-role-policy \
    --policy-arn $policy_arn \
    --role-name ${AWS::Region}-${AWS::AccountId}-SageMaker-OpenSearch-demo-role
```

You can see the custom policy below.  It allows the Amazon OpenSearch service to Invoke Amazon SageMaker endpoints as well as the Amazon Comprehend DetectDominantLanguage API

```
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Action": [
				"sagemaker:InvokeEndpointAsync",
				"sagemaker:InvokeEndpoint"
			],
			"Resource": [
				"arn:aws:sagemaker:__Region__:__AccountId__:endpoint/*"
			],
			"Effect": "Allow"
		},
        {
			"Action": [
                "comprehend:DetectDominantLanguage"
			],
			"Resource": "*",
			"Effect": "Allow"
		}
	]
}
```

Now, we can open up Cloudshell again to run our final command.

### Step 4 - Create a Sagemaker notebook instance
```
aws sagemaker create-notebook-instance \
    --notebook-instance-name ${AWS::Region}-${AWS::AccountId}-SageMaker-Execution-demo-notebook \
    --instance-type ml.m5.xlarge \
    --role-arn arn:aws:iam::${AWS::AccountId}:role/${AWS::Region}-${AWS::AccountId}-SageMaker-Execution-demo-role \
    --volume-size-in-gb 100 \
    --default-code-repository https://github.com/jtrollin/opensearch_sagemaker_multilingual
```

Once the Sagemaker notebook instance has been successfully created, navigate to the **SageMaker** dashboard in the console.

On the left hand menu click on the **Notebook** dropdown menu.
![notebook dashboard](images/notebooks.png)

Click on the **Notebook instances** link.
![notebook instances](images/demo_notebook.png)

Here you should see a notebook that has been created for you. Once the status of the notebook shows as **InService**  click on the **Open JupyterLab** link on the right.  This will launch the notebook.

On the right had side of the notebook, you will see a file directory.  Open the file named **OpensearchDemo.ipynb**.
![notebook instances](images/open_notebook.png)

The rest of this tutorial will be run from the notebook you just opened.
