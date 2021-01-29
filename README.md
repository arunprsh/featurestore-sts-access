## Using STS assumed roles to enable cross account access for SageMaker Feature Store

This document details the steps needed to enable cross account access for SageMaker Feature Store using an assumed role via AWS Security Token Service (**STS**). STS is a web service that enables you to request temporary, limited-privilege credentials for AWS Identity and Access Management (IAM) users. STS returns a set of temporary security credentials that you can use to access AWS resources that you might not normally have access to. These temporary credentials consist of an access key ID, a secret access key, and a security token.


### Short description:

To demonstrate this process, let us presume we have two accounts, A and B.

* Account B is the account that maintains a centralized feature store (Online and Offline stores).
* Account A is the account that needs access to both the Online and Offline store contained at account B. 

You can give SageMaker resources (notebooks, endpoints, training jobs)  created in account A permissions to assume a role from account B for the following reasons:

* To access resources, such as S3, Athena and Glue for cross account Offline store access.
* To allow cross account Online store access.

### Inside Account B:

* **Create an Assumed IAM Role:** This is the role which SageMaker resources in account A would assume to gain access to cross-account resources (in account B). 


Under *IAM → Create role →  Another AWS account*, provide the 12 digit account ID of account B as shown below:

[Image: image.png]
Hit Next: Permissions and under Permissions, search and attach the following AWS managed policies:


1. AmazonS3FullAccess
2. AmazonAthenaFullAccess
3. AmazonSageMakerFullAccess
4. Additionally, you would also have to create and attach a custom policy as shown below:


`{`
`  "Version": "2012-10-17",`
`  "Statement": [`
`    {`
`      "Sid": "AthenaResultsS3BucketCrossAccountAccessPolicy",`
`      "Effect": "Allow",`
`      "Action": [`
`        "s3:*"`
`      ],`
`      "Resource": [`
`        "arn:aws:s3:::<ATHENA RESULTS BUCKET NAME IN ACCOUNT A>/",`
`        "arn:aws:s3:::<ATHENA RESULTS BUCKET NAME IN ACCOUNT A>",`
`        "arn:aws:s3:::*SageMaker*"`
`      ]`
`    }`
`  ]`
`}`

`<ATHENA RESULTS BUCKET NAME IN ACCOUNT A>` is the bucket to which Athena query results will be written to. When we use the STS cross account role created above inside account A, it can run Athena queries against the Offline store content in account B without being in account B. The custom policy defined above allows Athena (inside account B) to write back the results to a results bucket in account A. Ensure this results bucket is created in account A before creating the policy above.

Once all the policies are attached, hit next and provide a name for this role. For this example, let us name it as `cross-account-assume-role`.

### **Inside Account A:**

* Create a SageMaker notebook instance with an IAM execution role. This role grants SageMaker notebook with the necessary permissions needed to execute actions on the feature store. By default, the following policies are attached when you create a new execution. See image below:

[Image: image.png]We need to create and attach the following policies in addition to the default ones mentioned above. 

* Custom policy as shown below which allows the execution role to perform certain S3 actions needed to interact with the feature store (Offline).

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "FeatureStoreS3AccessPolicy",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetBucketAcl",
        "s3:GetObjectAcl"
      ],
      "Resource": [
        "arn:aws:s3:::*SageMaker*"
      ]
    }
  ]
}
```

* Custom policy as show below which policy allows SageMaker notebook to assume the role(**cross-account-assume-role**) role created in account B.

```
{
    "Version": "2012-10-17",
    "Statement": {
        "Effect": "Allow",
        "Action": "sts:AssumeRole",
        "Resource": "arn:aws:iam::<ACCOUNT B ID>:role/cross-account-assume-role"
    }
}
```

* We know account A is the account that is enabled to cross access Online and Offline store in account B. When account A assumes the cross account STS role of B, it is capable of running Athena queries inside account B against the Offline store. The results of these queries (feature sets) however will need to be saved in account A’s S3 bucket in order to enable model training. Thus, we need to create a bucket in account A that can store the Athena query results as well as create a policy (shown below) to that bucket allowing cross account STS role to write and read objects to this bucket. 

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "MyStatementSid",
            "Effect": "Allow",
            "Principal": {
                "AWS": [
                    "arn:aws:iam::<ACCOUNT A ID>/<SAGEMAKER EXEC ROLE>",
                    "arn:aws:iam::<ACCOUNT A>:root",
                    "arn:aws:iam::<ACCOUNT B>:role/cross-account-assume-role",
                    "arn:aws:iam::<ACCOUNT B>:root"
                ]
            },
            "Action": "s3:*",
            "Resource": [
                "arn:aws:s3:::<ATHENA RESULTS BUCKET NAME IN ACCOUNT A>",
                "arn:aws:s3:::<ATHENA RESULTS BUCKET NAME IN ACCOUNT A>/*"
            ]
        }
    ]
}
```

### **Inside Account B:**

Now, since we had created an execution role in account A, use the ARN of this role to modify the trust policy of the cross account assume role in account A. Also, add SageMaker service as a principal to the policy.


```
`{`
`    ``"Version"``:`` ``"2012-10-17"``,`
`    ``"Statement"``:`` ``[`
`        ``{`
`            ``"Effect"``:`` ``"Allow"``,`
`            ``"Principal"``:`` ``{
                "Service": "sagemaker.amazonaws.com",`
`                ``"AWS"``:`` ``"ARN OF SAGEMAKER EXECUTION ROLE CREATED IN ACCOUNT A"`
`            ``},`
`            ``"Action"``:`` ``"sts:AssumeRole"`
`        ``}`
`    ``]`
`}`
```



