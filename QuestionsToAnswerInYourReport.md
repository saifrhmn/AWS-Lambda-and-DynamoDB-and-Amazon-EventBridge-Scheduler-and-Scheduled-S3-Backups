1. Question: What is the difference between Lambda's execution role and Scheduler's execution role?  
   Answer: Lambda's role lets the function access AWS resources; Scheduler's role lets Scheduler invoke the function. *Example: Lambda #2's role writes to S3; Scheduler's role starts Lambda #2.*
2. Question: Why doesn't running Lambda #1 twice produce 20 DynamoDB users?  
   Answer: DynamoDB `PutItem` replaces an item with the same key instead of adding a duplicate. *Example: writing user `id=1` twice leaves one item with `id=1`.*
3. Question: Why do Lambda #2 backups have unique timestamps and request IDs?  
   Answer: Lambda #2 adds the current timestamp and invocation request ID to each S3 key. *Example: `backups/2026-10-27/users-11-30-00-123456-abc123.json`.*
4. Question: What is the difference between `print()` logs and `Invocations`/`Errors` metrics?  
   Answer: `print()` creates detailed log messages; `Invocations` and `Errors` are CloudWatch counts. *Example: “Saved user 1” is a log; one successful run adds one invocation and zero errors.*
5. Question: What happens if you delete the DynamoDB table but keep its JSON files in S3? Can the table's indexes/configuration be restored from the JSON alone?  
   Answer: The JSON files keep the exported items, but not the table's indexes or configuration. You can recreate a table and import the items, but must restore its schema and indexes separately.
6. Question: Why is exporting every 2 minutes usually a poor production backup strategy for large tables?  
   Answer: Frequent full scans can be slow, costly, and consume read capacity; a scan also is not a point-in-time backup. *Example: a large table may take longer than the Lambda timeout to scan.* Use PITR or managed DynamoDB Export to S3 for production recovery.