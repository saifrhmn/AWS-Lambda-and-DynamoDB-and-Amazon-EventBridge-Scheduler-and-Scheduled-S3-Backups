# Scheduled DynamoDB JSON Export to Amazon S3

In this assignment I built a small serverless backup system on AWS. One Lambda function loads sample users into DynamoDB. A second Lambda function reads the table and saves it as a JSON file in S3. EventBridge Scheduler runs the second function every 2 minutes, and CloudWatch keeps the logs.

> **Region I used:** Asia Pacific (Singapore) - `ap-southeast-1`. I created the DynamoDB table, both Lambda functions, the S3 bucket and the schedule in this Region.

## How it works

![AWS Lambda, DynamoDB, EventBridge Scheduler, and S3 architecture](<ScreenShots/Screenshot # Architecture.png>)

1. `import-jsonplaceholder-users` downloads 10 sample users from JSONPlaceholder and saves them in the `jsonplaceholder-users` DynamoDB table.
2. `dynamodb-backup-every-2min` is an EventBridge Scheduler schedule that invokes `export-dynamodb-to-s3` every 2 minutes.
3. `export-dynamodb-to-s3` scans the table and saves a timestamped JSON file in my bucket under `backups/YYYY-MM-DD/`.
4. Both functions write their logs to CloudWatch.

## How I completed the assignment

### Step 1: Prepared the Lambda code

I wrote the two functions in Python:

- Import users: [lambda_src/import_users/lambda_function.py](lambda_src/import_users/lambda_function.py)
- Export users to S3: [lambda_src/backup_users/lambda_function.py](lambda_src/backup_users/lambda_function.py)

Then I packaged each one as a ZIP, with `lambda_function.py` at the root of the ZIP:

```powershell
Compress-Archive -Path .\lambda_src\import_users\lambda_function.py `
  -DestinationPath .\lambda_zips\lambda-import-users.zip -Force

Compress-Archive -Path .\lambda_src\backup_users\lambda_function.py `
  -DestinationPath .\lambda_zips\lambda-backup-to-s3.zip -Force
```

### Step 2: Created the DynamoDB table

I created the table `jsonplaceholder-users` with partition key `id` (Number) and on-demand capacity. It was empty at first.

![DynamoDB table jsonplaceholder-users with id (Number) partition key](<s3 srn-shot/s3 srn-shot/dynamic-table.png>)

### Step 3: Created the S3 bucket

I created a private bucket named `student-lambda-backups-saifur` in `ap-southeast-1`.

![S3 bucket student-lambda-backups-saifur](<s3 srn-shot/s3 srn-shot/s3-bucket.png>)

### Step 4: Gave each function its own IAM role

Each Lambda function has its own execution role. Both roles have the AWS-managed `AWSLambdaBasicExecutionRole` policy so they can write logs to CloudWatch.

- The import role has an inline policy that allows only `dynamodb:PutItem` on the table, based on [iam_policies/lambda-import-putitem-policy.json](iam_policies/lambda-import-putitem-policy.json).
- The export role allows `dynamodb:Scan` on the table and `s3:PutObject` on the bucket, based on [iam_policies/lambda-backup-export-policy.json](iam_policies/lambda-backup-export-policy.json).

![Import function role with AllowPutDemoUsers and AWSLambdaBasicExecutionRole](<s3 srn-shot/s3 srn-shot/IAM-json.png>)

### Step 5: Created the Lambda functions

I created both functions with a Python runtime, uploaded the ZIP files and set the handler to `lambda_function.lambda_handler`.

| Function | ZIP package | Environment variables |
| --- | --- | --- |
| `import-jsonplaceholder-users` | `lambda-import-users.zip` | `TABLE_NAME=jsonplaceholder-users` |
| `export-dynamodb-to-s3` | `lambda-backup-to-s3.zip` | `TABLE_NAME=jsonplaceholder-users`, `BUCKET_NAME=student-lambda-backups-saifur`, `BACKUP_PREFIX=backups` |

![import-jsonplaceholder-users Lambda function](<s3 srn-shot/s3 srn-shot/lamda-fucntion-1.png>)

### Step 6: Ran the import function

I tested `import-jsonplaceholder-users` with an empty event `{}`. It returned status 200 and `saved_count: 10`, so all 10 users were saved in DynamoDB. Running it again does not create duplicates, because `PutItem` replaces items that have the same `id`.

![Import test succeeded with saved_count 10](<s3 srn-shot/s3 srn-shot/import-json-10 -items.png>)

### Step 7: Ran the export function by hand once

Before setting up the schedule, I tested `export-dynamodb-to-s3` manually with `{}`. It returned status 200, `exported_count: 10` and the S3 path of the new file.

![Export test succeeded with exported_count 10](<s3 srn-shot/s3 srn-shot/export-user-sucss-resilt.png>)

At this point the `backups/2026-11-10/` folder had just one file, created by this manual test at 22:24:40 (UTC+6).

![One backup file in S3 after the manual test](<s3 srn-shot/s3 srn-shot/s3-backup-update.png>)

### Step 8: Created the schedule

In EventBridge Scheduler I created `dynamodb-backup-every-2min`:

- Recurring schedule: `rate(2 minutes)`
- Flexible time window: off
- Target: `export-dynamodb-to-s3` (Lambda Invoke)
- Execution role: a Scheduler role that is allowed to invoke this Lambda function

The schedule was created and **Enabled** on Nov 10, 2026 at 22:36:34 (UTC+6).

![Schedule dynamodb-backup-every-2min created with rate(2 minutes)](<s3 srn-shot/s3 srn-shot/even-bridge-created.png>)

![Schedule list showing dynamodb-backup-every-2min Enabled with target export-dynamodb-to-s3](<s3 srn-shot/s3 srn-shot/event-bridge-enabled.png>)

### Step 9: Proved the schedule runs the backup automatically

After enabling the schedule I did **not** press Test. I waited and checked S3. New backup files appeared on their own, about every 2 minutes:

| File time (UTC+6) | Gap |
| --- | --- |
| 22:37:18 | first run after the schedule was enabled |
| 22:38:53 | ~2 min |
| 22:40:53 | 2 min |
| 22:42:53 | 2 min |
| 22:44:53 | 2 min |
| 22:46:53 | 2 min |
| 22:48:53 | 2 min |

The steady 2-minute gap shows that EventBridge Scheduler was starting the Lambda function, not me.

![S3 backups created every 2 minutes by the schedule](<s3 srn-shot/s3 srn-shot/backup-with-2mins.png>)

Later the folder had 51 backup files.

![S3 backup folder with 51 objects](<s3 srn-shot/s3 srn-shot/log-s3-backup.png>)

### Step 10: Checked the CloudWatch logs

The log group `/aws/lambda/export-dynamodb-to-s3` shows each run. A successful run reads the table, reads 10 items in 1 scan page, and prints `SUCCESS: Exported 10 users to s3://student-lambda-backups-saifur/backups/...`. The file name in the log matches a file in the S3 folder (`users-19-39-14-284716-...json`).

![CloudWatch log showing SUCCESS: Exported 10 users to S3](<s3 srn-shot/s3 srn-shot/evenbrige-log.png>)

## Answers to the report questions

My answers are in [QuestionsToAnswerInYourReport.md](QuestionsToAnswerInYourReport.md).

## Clean up

After capturing the scheduled-backup evidence, I disabled the `dynamodb-backup-every-2min` schedule on November 11, 2026. It is disabled, not deleted, so it can be re-enabled if needed. The screenshots show its status before and after cleanup:

- [Schedule enabled before cleanup](Scheduled-Backup-README-Screenshots/eventbridge-schedule-enabled-before-cleanup.png)
- [Schedule disabled after cleanup](Scheduled-Backup-README-Screenshots/eventbridge-schedule-disabled-after-cleanup.png)

The Lambda functions, DynamoDB table, S3 bucket, and IAM roles were not deleted. Delete those resources only if they are no longer needed.
