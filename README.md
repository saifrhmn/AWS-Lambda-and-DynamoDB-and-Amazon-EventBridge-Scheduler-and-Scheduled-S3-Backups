# Scheduled DynamoDB JSON Export to Amazon S3

This project uses two AWS Lambda functions to import sample users into DynamoDB and export them as JSON files to an S3 bucket. EventBridge Scheduler can run the export function automatically, and CloudWatch stores the function logs.

> **AWS Region:** US East (N. Virginia) - `us-east-1`. Create the DynamoDB table, Lambda functions, and EventBridge schedule in this same Region. S3 bucket names are global and must be unique.

## How it works

![AWS Lambda, DynamoDB, EventBridge Scheduler, and S3 architecture](<ScreenShots/Screenshot # Architecture.png>)

1. `import-jsonplaceholder-users` gets sample users from JSONPlaceholder and saves them in the `jsonplaceholder-users` DynamoDB table.
2. `dynamodb-backup-every-2min` is an EventBridge Scheduler schedule that invokes `export-dynamodb-to-s3`.
3. `export-dynamodb-to-s3` reads the table and saves a timestamped JSON file to the S3 `backups/` folder.
4. Both functions send logs to CloudWatch.

## Develop and run the project

### 1. Edit and package the Lambda code

The function source files are:

- Import users: `lambda_src/import_users/lambda_function.py`
- Export users to S3: `lambda_src/backup_users/lambda_function.py`

Edit the relevant Python file to change a function. From the repository folder, package each function before uploading it. The ZIP must contain `lambda_function.py` at its root.

```powershell
Compress-Archive -Path .\lambda_src\import_users\lambda_function.py `
  -DestinationPath .\lambda_zips\lambda-import-users.zip -Force

Compress-Archive -Path .\lambda_src\backup_users\lambda_function.py `
  -DestinationPath .\lambda_zips\lambda-backup-to-s3.zip -Force
```

### 2. Create the DynamoDB table and S3 bucket

In the AWS Console, switch to **US East (N. Virginia) (`us-east-1`)**:

1. Create a DynamoDB table named `jsonplaceholder-users`. Set its partition key to `id` with type **Number**, and select on-demand capacity.
2. Create a private S3 bucket with a globally unique name. Use your bucket name in place of `YOUR_BUCKET` below.

![DynamoDB table with numeric id partition key](<ScreenShots/Screenshot # 1.png>)

### 3. Create permissions for the Lambda functions

Create a separate Lambda execution role for each function. Add the AWS-managed `AWSLambdaBasicExecutionRole` policy to both roles for CloudWatch logging.

- Import role: use [iam_policies/lambda-import-putitem-policy.json](iam_policies/lambda-import-putitem-policy.json). Replace `REGION` and `ACCOUNT_ID`.
- Export role: use [iam_policies/lambda-backup-export-policy.json](iam_policies/lambda-backup-export-policy.json). Replace `REGION`, `ACCOUNT_ID`, and `YOUR_UNIQUE_BUCKET_NAME`.

Use `us-east-1` for `REGION`. The Scheduler will use a separate role to invoke the export function.

### 4. Create and configure the Lambda functions

Create both functions in the Lambda Console using a supported Python runtime and their corresponding execution roles. Upload the ZIP packages from `lambda_zips/` and set the handler to `lambda_function.lambda_handler`.

| Function | ZIP package | Environment variables |
| --- | --- | --- |
| `import-jsonplaceholder-users` | `lambda-import-users.zip` | `TABLE_NAME=jsonplaceholder-users` |
| `export-dynamodb-to-s3` | `lambda-backup-to-s3.zip` | `TABLE_NAME=jsonplaceholder-users`, `BUCKET_NAME=YOUR_BUCKET`, `BACKUP_PREFIX=backups` (optional) |

### 5. Run the import function

In `import-jsonplaceholder-users`, create a test event with `{}` and choose **Test**. A successful response reports the number of users saved. Check the DynamoDB table to see the imported items. Running the import again replaces items with the same `id`; it does not create duplicates.

![Successful Lambda import test](<ScreenShots/Screenshot #1 [DynamicDB].png>)

### 6. Run the export function

In `export-dynamodb-to-s3`, create a test event with `{}` and choose **Test**. A successful response includes the S3 URI. Open your bucket and look under `backups/YYYY-MM-DD/` to find the exported JSON file.

![Successful Lambda export test](<ScreenShots/Screenshot # 5.png>)

![Exported JSON backups in the S3 bucket](<ScreenShots/Screenshot # 8.png>)

If a test fails, open the function's CloudWatch log group and check the error. The import role needs `dynamodb:PutItem`; the export role needs `dynamodb:Scan` and `s3:PutObject`.

### 7. Schedule automatic exports

In EventBridge Scheduler, create a schedule named `dynamodb-backup-every-2min`:

1. Choose a recurring schedule with the expression `rate(2 minutes)` and turn off the flexible time window.
2. Set `export-dynamodb-to-s3` as the target.
3. Choose or create a Scheduler execution role that allows EventBridge Scheduler to invoke this Lambda function.
4. Create the schedule and confirm it is **Enabled**.

![Example EventBridge Scheduler configuration](<ScreenShots/Screenshot # 7.png>)

The screenshot is an example of a previous configuration; the schedule was later deleted. Create a new schedule in your account to run this project. Wait for scheduled runs and check CloudWatch and S3 rather than manually testing the function when verifying automation.

### 8. MISSING: Capture proof of an automatic scheduled backup

**This part of the assignment is not finished yet.** The screenshots below are old examples. They show that a backup worked when the Lambda was tested, but they do not prove that the schedule ran it automatically.

#### Old examples: where to look

**Schedule settings:** The schedule should say **Enabled** and show `rate(2 minutes)` and the `export-dynamodb-to-s3` function. This old screenshot is only an example:

![Old example of the EventBridge Scheduler details](<ScreenShots/Screenshot # 7.png>)

**Lambda log:** A successful run has a message beginning `SUCCESS: Exported ... users to s3://...`. This old screenshot shows a successful export, but not what started it:

![Old example of a successful export log](<ScreenShots/Screenshot # 5 log.png>)

**S3 backup files:** New JSON files should appear in the bucket. This old screenshot is not linked to a confirmed scheduled run:

![Old example of JSON objects in the S3 backup folder](<ScreenShots/Screenshot # 8.png>)

#### What to do for the assignment

1. In EventBridge Scheduler, create or enable the schedule. Make sure it is **Enabled**, runs every two minutes, and targets `export-dynamodb-to-s3`.
2. Take a screenshot of these schedule settings.
3. Wait at least four minutes. **Do not press Test or run the Lambda yourself.** Let the schedule start it.
4. Take a screenshot of the Lambda's CloudWatch logs showing a successful export after you enabled the schedule.
5. Take a screenshot of the S3 bucket showing the new JSON backup file or files.
6. Add the new screenshots to `ScreenShots/` and link them here. The times on the schedule, log, and S3 screenshots should be from the same period.

You need these three screenshots: **enabled schedule**, **successful Lambda log**, and **new S3 backup file(s)**. Together, they show that the schedule ran the backup. A Lambda test screenshot by itself only shows a manual run. If available, also include a screenshot of the Scheduler's run count for the same period.

**Example:** The schedule is enabled at 10:00. You do not press Test. At about 10:02, the Lambda log shows that it saved a file, and S3 shows that file with a matching time. Take screenshots of all three.

### 9. Clean up

Disable or delete the schedule when finished to prevent further invocations and charges. Delete test resources from your AWS account if you no longer need them.
