# Creating an Amazon S3 and Databricks Pipeline

Connect an S3 bucket to Databricks Unity Catalog with a storage credential and an external location. You need permission to create IAM policies and roles in AWS, and metastore admin privileges in Databricks.

Use your own names, or the examples below:

- Bucket: `ipl-analyst-data-project`
- Storage credential: `ipl_analyst_cred`
- External location: `ipl_analyst_loc`

## Step 1 — Create an S3 bucket and load your data

Create the bucket in AWS, then upload the files you want Databricks to read.

## Step 2 — Create an IAM policy for the bucket

In the IAM console, create a policy with the permissions below. Replace `<BUCKET_NAME>` with your bucket name.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:GetObjectVersion",
        "s3:ListBucket",
        "s3:GetBucketLocation"
      ],
      "Resource": [
        "arn:aws:s3:::<BUCKET_NAME>",
        "arn:aws:s3:::<BUCKET_NAME>/*"
      ]
    }
  ]
}
```

Add `s3:PutObject` and `s3:DeleteObject` if you also want to write data back to the bucket.

## Step 3 — Create an IAM role and attach the policy

Create an IAM role, attach the policy from Step 2, and use the starter trust policy below so AWS can save the role. You will replace this trust policy in Step 5.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::414351767826:role/unity-catalog-prod-UCMasterRole-14S5ZJVKOTYTL"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

Copy the role ARN. It looks like this:

```text
arn:aws:iam::<YOUR-AWS-ACCOUNT-ID>:role/<THIS-ROLE-NAME>
```

## Step 4 — Create a storage credential in Databricks

1. Go to Catalog Explorer → External Locations → Storage Credentials → Create.
2. Select AWS IAM role.
3. Name it, for example `ipl_analyst_cred`.
4. Paste the role ARN from Step 3.
5. Click Create.

Copy the external ID Databricks shows for this credential. You need it in the next step.

## Step 5 — Update the IAM role trust policy

In AWS, open the role from Step 3 and replace its trust policy with the policy below.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": [
          "arn:aws:iam::414351767826:role/unity-catalog-prod-UCMasterRole-14S5ZJVKOTYTL",
          "arn:aws:iam::<YOUR-AWS-ACCOUNT-ID>:role/<THIS-ROLE-NAME>"
        ]
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "<STORAGE-CREDENTIAL-EXTERNAL-ID>"
        }
      }
    }
  ]
}
```

Replace these three values:

- `<YOUR-AWS-ACCOUNT-ID>` — your 12-digit AWS account ID
- `<THIS-ROLE-NAME>` — the name of the IAM role from Step 3
- `<STORAGE-CREDENTIAL-EXTERNAL-ID>` — the external ID you copied in Step 4

## Step 6 — Create an external location

1. Go to Catalog Explorer → External Locations → Create.
2. Set the URL to `s3://<BUCKET_NAME>/`, for example `s3://ipl-analyst-data-project/`.
3. Select the storage credential from Step 4, for example `ipl_analyst_cred`.
4. Name the external location, for example `ipl_analyst_loc`.
5. Click Create.

You can create the same location with SQL instead:

```sql
CREATE EXTERNAL LOCATION IF NOT EXISTS <EXTERNAL_LOCATION_NAME>
URL 's3://<BUCKET_NAME>/'
WITH (STORAGE CREDENTIAL <CREDENTIAL_NAME>)
COMMENT 'External location for the S3 bucket';
```

## Step 7 — Grant yourself access

On the external location page, open Permissions and grant your user:

- `READ FILES`, so you can read the data
- `WRITE FILES`, if you want to write data back
- `CREATE EXTERNAL VOLUME`, if you want to create volumes on this location

Or run this SQL in a Databricks notebook. Replace the email with your admin user.

```sql
GRANT READ FILES ON EXTERNAL LOCATION <EXTERNAL_LOCATION_NAME> TO `<YOUR_ADMIN_EMAIL>`;
GRANT WRITE FILES ON EXTERNAL LOCATION <EXTERNAL_LOCATION_NAME> TO `<YOUR_ADMIN_EMAIL>`;
GRANT CREATE EXTERNAL VOLUME ON EXTERNAL LOCATION <EXTERNAL_LOCATION_NAME> TO `<YOUR_ADMIN_EMAIL>`;
```
