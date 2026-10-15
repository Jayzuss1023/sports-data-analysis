# Creating an Amazon S3 and Databricks pipeline

This is the first step in the [IPL data analysis](../README.md). It connects an S3 bucket to Databricks so the notebook can read the CSV files through Unity Catalog. You need permission to create IAM policies and roles in AWS, and metastore admin privileges in Databricks.

An IAM policy does not produce a role ARN, and Databricks does not show the external ID until the storage credential exists. Create the bucket and the IAM role first, create the storage credential next, then update the role's trust policy, and only then create the external location. The location is created once. The SQL at the end is another way to do that same step, not a second location.

When this setup is done, continue with [Loading the tables](loading-data.md).

Use your own names, or the examples below:

| Piece | Example |
| --- | --- |
| Bucket | `ipl-analyst-data-project` |
| IAM role | `ipl-analyst-role` |
| Storage credential | `ipl_analyst_cred` |
| External location | `ipl_analyst_loc` |

## Step 1. Create the S3 bucket and load the data

1. In the AWS console, open S3 and choose **Create bucket**.
2. Name the bucket. Names are global, so `ipl-analyst-data-project` fails if someone else already took it. Pick another name and use that name everywhere below.
3. Choose a region close to your Databricks workspace. Leave **Block all public access** on. Databricks reaches the files through the IAM role, not through a public URL.
4. Create the bucket, open it, and upload the five CSV files at the bucket root (not inside a folder, unless you add that folder to the paths in the notebook):

- `Ball_By_Ball.csv`
- `Match.csv`
- `Player.csv`
- `Player_match.csv`
- `Team.csv`

The notebook currently reads `Ball_By_Ball.csv` from `s3://ipl-analyst-data-project/` and the other four files from `s3://ipl-data-analysis-project/`. Put every file in the bucket you just created, then point all five paths in [Loading the tables](loading-data.md) at that one bucket.

Default encryption (SSE-S3) is enough. If you switch the bucket to a customer-managed KMS key, also allow `kms:Decrypt` on that key in the policy in the next step.

## Step 2. Create an IAM policy for the bucket

This is an IAM policy, not an S3 bucket policy. In the IAM console, go to **Policies**, choose **Create policy**, open the JSON tab, and paste the policy below. Replace `<BUCKET_NAME>` with your bucket name.

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

`s3:GetBucketLocation` is what Unity Catalog uses to confirm which region the bucket is in. Add `s3:PutObject` and `s3:DeleteObject` if you also want to write data back to the bucket.

Name the policy (for example `ipl-analyst-s3-read`) and create it. Copying a role ARN here will not work. The policy only grants S3 access. The role ARN comes from the next step.

## Step 3. Create an IAM role and attach the policy

In IAM, go to **Roles** and choose **Create role**. Use **Custom trust policy** and paste the starter policy below. This lets AWS save the role before you know the external ID. You replace this trust policy in Step 5.

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

`414351767826` is Databricks' AWS account, not yours. That role is the Unity Catalog master role Databricks uses to assume roles in your account. If the storage-credential page in your workspace shows a different master-role ARN, use the one Databricks shows.

On the permissions step, attach the policy from Step 2. Name the role (for example `ipl-analyst-role`) and create it.

Open the role and copy its ARN. It looks like this:

```text
arn:aws:iam::<YOUR-AWS-ACCOUNT-ID>:role/<THIS-ROLE-NAME>
```

Your 12-digit account ID is in the account menu at the top right of the AWS console.

## Step 4. Create a storage credential in Databricks

This requires metastore admin privileges (or `CREATE STORAGE CREDENTIAL` on the metastore).

1. Go to **Catalog Explorer**, then **External Locations**, then **Storage Credentials**, then **Create**. In some workspaces the same page is under **Catalog**, then **External data**.
2. Select **AWS IAM role**.
3. Name it, for example `ipl_analyst_cred`.
4. Paste the role ARN from Step 3: `arn:aws:iam::<YOUR-AWS-ACCOUNT-ID>:role/<THIS-ROLE-NAME>`.
5. Click **Create**.

Open the credential and copy the **external ID**. Databricks generates it when the credential is created. You need that value in the next step. It does not come from the IAM policy.

## Step 5. Update the IAM role trust policy

Back in AWS, open the role from Step 3, go to **Trust relationships**, and choose **Edit trust policy**. Replace the starter policy with the one below.

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

- `<YOUR-AWS-ACCOUNT-ID>`: your 12-digit AWS account ID
- `<THIS-ROLE-NAME>`: the name of the IAM role from Step 3
- `<STORAGE-CREDENTIAL-EXTERNAL-ID>`: the external ID you copied from the storage credential in Step 4

The first principal is Databricks. The second is the role itself. Unity Catalog requires that self-reference so the role can assume itself after Databricks hands it off. The external ID stops another account from reusing this role. Paste the JSON with straight quotes. Curly quotes from a notes doc make IAM reject the policy.

IAM can take a minute to apply the new trust policy. If the next step fails immediately with an assume-role error, wait and try again.

## Step 6. Create an external location

The storage credential says *which role* can reach AWS. The external location says *which path* that role may read.

1. Go to **Catalog Explorer**, then **External Locations**, then **Create**.
2. Set the URL to `s3://<BUCKET_NAME>/`, for example `s3://ipl-analyst-data-project/`. Include the trailing slash.
3. Select the storage credential from Step 4, for example `ipl_analyst_cred`.
4. Name the external location, for example `ipl_analyst_loc`.
5. Click **Create**.

You can create the same location with SQL in a notebook instead of the UI. Run it once. `<CREDENTIAL_NAME>` is the storage credential, not the IAM role name.

```sql
CREATE EXTERNAL LOCATION IF NOT EXISTS <EXTERNAL_LOCATION_NAME>
URL 's3://<BUCKET_NAME>/'
WITH (STORAGE CREDENTIAL <CREDENTIAL_NAME>)
COMMENT 'External location for the S3 bucket';
```

For the example names:

```sql
CREATE EXTERNAL LOCATION IF NOT EXISTS ipl_analyst_loc
URL 's3://ipl-analyst-data-project/'
WITH (STORAGE CREDENTIAL ipl_analyst_cred)
COMMENT 'External location for the S3 bucket';
```

## Step 7. Grant yourself access

On the external location page, open **Permissions** and grant your user:

- **READ FILES**, so you can read the data
- **WRITE FILES**, if you want to write data back
- **CREATE EXTERNAL VOLUME**, if you want to create volumes on this location

There is no `USE` privilege on an external location. Creating a volume uses `CREATE EXTERNAL VOLUME`.

Or run this SQL in a Databricks notebook. The backticks around the email are required. Replace the email with your admin user.

```sql
GRANT READ FILES ON EXTERNAL LOCATION <EXTERNAL_LOCATION_NAME> TO `<YOUR_ADMIN_EMAIL>`;
GRANT WRITE FILES ON EXTERNAL LOCATION <EXTERNAL_LOCATION_NAME> TO `<YOUR_ADMIN_EMAIL>`;
GRANT CREATE EXTERNAL VOLUME ON EXTERNAL LOCATION <EXTERNAL_LOCATION_NAME> TO `<YOUR_ADMIN_EMAIL>`;
```

## Check that the path is readable

In a notebook attached to a Unity Catalog-enabled cluster:

```sql
LIST 's3://<BUCKET_NAME>/';
```

You should see the five CSV files. A read of the first few rows is the same check the notebook uses later:

```python
(
    spark.read.format("csv")
    .option("header", "true")
    .load("s3://<BUCKET_NAME>/Ball_By_Ball.csv")
    .limit(5)
    .show()
)
```

If `LIST` fails:

- **AssumeRole / external ID errors** mean Step 5 still has the starter trust policy, the wrong external ID, or a missing self-reference.
- **AccessDenied on s3** means the IAM policy bucket name does not match the URL, or the policy is not attached to the role.
- **Permission denied in Databricks** means Step 7 did not grant `READ FILES` to the user running the notebook.

Next: [Loading the tables](loading-data.md).
