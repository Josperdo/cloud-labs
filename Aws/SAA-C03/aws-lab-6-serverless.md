# AWS Lab 6: Serverless Architecture

Check box if done: []

## Overview
Everything so far has been "keep servers running and make them resilient." This lab flips the model: no servers to patch, no Auto Scaling Group to size, no AZ failover to plan for — Lambda, API Gateway, and DynamoDB handle all of that for you. You'll build a small REST API end to end and then focus on the part that actually matters for the exam and for interviews: IAM execution roles scoped to exactly what each function needs, and nothing more.

**Estimated time**: 3–4 hours
**Cost**: ~$0 (Lambda, API Gateway, and DynamoDB all have generous always-free tiers well beyond what this lab uses)

---

## Objectives
- Build a DynamoDB table and understand partition keys
- Write and deploy a Lambda function with a least-privilege execution role
- Expose the function through API Gateway as a REST endpoint
- Understand cold starts, concurrency, and where serverless stops being the right choice
- Compare the operational model directly against Lab 5's always-on architecture

---

## Part 1: DynamoDB Table

### Step 1: Create the Table
1. **DynamoDB** → **Create table**
2. **Table name**: `tasks`
3. **Partition key**: `taskId` (String)
4. **Table settings**: `Default settings` (on-demand capacity — no capacity units to provision or size, matching the serverless theme)
5. **Create table**

### Step 2: Understand the Partition Key Choice
1. **DynamoDB** has no concept of foreign keys, joins, or a query planner like a relational database — every access pattern has to be designed around the partition key up front
2. For this lab, `taskId` (a random UUID per task) is fine for simple lookups. In a real design, you'd choose the partition key based on your dominant query pattern (e.g., `userId` if "get all tasks for a user" is the most common query) — this is a standard SAA-C03 exam scenario, so note it even though this lab doesn't need that complexity

---

## Part 2: Lambda Function with a Least-Privilege Role

### Step 3: Create the Execution Role First (Before the Function)
Building the role first — rather than accepting Lambda's default "create a new role with basic permissions" and editing it after — forces you to think about exactly what the function needs.

1. **IAM** → **Roles** → **Create role**
2. **Trusted entity type**: `AWS service` → **Lambda**
3. Skip attaching a managed policy for now — you'll attach a custom inline policy instead
4. **Role name**: `tasks-lambda-execution-role`
5. **Create role**

### Step 4: Attach a Custom Least-Privilege Policy
1. Open `tasks-lambda-execution-role` → **Add permissions** → **Create inline policy** → **JSON**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:PutItem",
        "dynamodb:GetItem",
        "dynamodb:Query"
      ],
      "Resource": "arn:aws:dynamodb:*:*:table/tasks"
    },
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:*"
    }
  ]
}
```
2. **Name**: `tasks-dynamodb-least-privilege` → **Create policy**

**What this avoids**: the common shortcut of attaching `AmazonDynamoDBFullAccess` (every table, every action, including `DeleteTable`) to a function that only ever needs to put and get items from one specific table. If this function is ever compromised via a code vulnerability, the blast radius is exactly these three actions on this one table — not your entire DynamoDB estate.

### Step 5: Create the Lambda Function
1. **Lambda** → **Create function**
2. **Function name**: `tasks-api`
3. **Runtime**: `Python 3.12`
4. **Permissions** → **Use an existing role** → `tasks-lambda-execution-role`
5. **Create function**

### Step 6: Write the Function Code
Paste into the inline code editor:

```python
import json
import uuid
import boto3

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('tasks')

def lambda_handler(event, context):
    method = event.get('httpMethod', 'POST')

    if method == 'POST':
        body = json.loads(event.get('body') or '{}')
        task_id = str(uuid.uuid4())
        table.put_item(Item={
            'taskId': task_id,
            'description': body.get('description', ''),
            'done': False
        })
        return {
            'statusCode': 201,
            'body': json.dumps({'taskId': task_id})
        }

    if method == 'GET':
        task_id = event.get('pathParameters', {}).get('taskId')
        response = table.get_item(Key={'taskId': task_id})
        item = response.get('Item')
        if not item:
            return {'statusCode': 404, 'body': json.dumps({'error': 'not found'})}
        return {'statusCode': 200, 'body': json.dumps(item)}

    return {'statusCode': 405, 'body': json.dumps({'error': 'method not allowed'})}
```

3. **Deploy**

### Step 7: Test Directly in the Lambda Console First
Before wiring up API Gateway, test the function in isolation — isolate whether a bug is in the Lambda logic or the API Gateway integration.

1. **Test** tab → **Create new event**:
```json
{ "httpMethod": "POST", "body": "{\"description\": \"Finish AZ-500 labs\"}" }
```
2. **Test** → confirm a `201` response with a generated `taskId`

**Validation checkpoint**: **DynamoDB** → `tasks` table → **Explore table items** — confirm the item you just created appears with the matching `taskId`.

---

## Part 3: Expose via API Gateway

### Step 8: Create a REST API
1. **API Gateway** → **Create API** → **REST API** → **Build**
2. **API name**: `tasks-api-gateway`
3. **Create API**

### Step 9: Create Resources and Methods
1. **Actions** → **Create Resource** → **Resource Name**: `tasks`
2. On `/tasks` → **Actions** → **Create Method** → `POST` → **Integration type**: `Lambda Function` → select `tasks-api`
3. Under `/tasks`, **Create Resource** → **Resource Name**: `{taskId}` (this creates a path parameter)
4. On `/tasks/{taskId}` → **Create Method** → `GET` → Lambda integration → `tasks-api`

### Step 10: Deploy the API
1. **Actions** → **Deploy API** → **Deployment stage**: `[New Stage]` → **Stage name**: `dev`
2. Note the **Invoke URL** shown at the top of the stage editor

### Step 11: Test End to End
```bash
INVOKE_URL="<your-invoke-url>"

curl -X POST "$INVOKE_URL/tasks" \
  -H "Content-Type: application/json" \
  -d '{"description": "Test from curl"}'
# Note the returned taskId

curl "$INVOKE_URL/tasks/<taskId-from-above>"
```

**Validation checkpoint**: the `GET` request returns the same task you just created, proving the full path — API Gateway → Lambda → DynamoDB, using only the least-privilege role from Step 4 — works end to end.

---

## Part 4: Cold Starts, Concurrency, and When Serverless Isn't the Right Fit

### Step 12: Observe a Cold Start
1. **Lambda** → `tasks-api` → **Monitor** → **View CloudWatch logs**
2. Invoke the function after it's been idle for 10+ minutes and check the log's `REPORT` line — note the `Init Duration` field. This is the cold-start cost: AWS provisioning a fresh execution environment before your code runs
3. Invoke it again immediately — the second invocation reuses the warm environment, and `Init Duration` won't appear at all

### Step 13: Understand the Tradeoffs (Interview Framing)
- **Serverless wins when**: traffic is spiky or unpredictable, you don't want to manage patching/scaling, and occasional cold-start latency (tens to low hundreds of milliseconds for a small Python function) is acceptable
- **Always-on (Lab 5) wins when**: you need consistent low latency on every single request, you're running long-lived connections or background processes Lambda's execution time limits don't fit well, or you have steady, predictable, high-volume traffic where reserved/always-on compute is more cost-effective than per-invocation billing

---

## Part 5: Cleanup

```bash
aws dynamodb delete-table --table-name tasks
aws lambda delete-function --function-name tasks-api
```

Then in the console:
1. **API Gateway** → delete `tasks-api-gateway`
2. **IAM** → **Roles** → delete `tasks-lambda-execution-role`

```bash
aws dynamodb describe-table --table-name tasks 2>&1 | grep -i "not found\|ResourceNotFoundException" && echo "Cleaned up"
```

---

## Key Concepts to Understand

| Concept | Definition |
|---------|-----------|
| **Execution role vs. resource policy** | The execution role controls what the *function* can do to other AWS services; a resource policy on the function controls who can *invoke* it — different directions, easy to conflate |
| **Cold start** | Latency incurred when Lambda provisions a fresh execution environment; only affects the first invocation after idle time or a scale-up event |
| **On-demand vs. provisioned DynamoDB capacity** | On-demand bills per request with no capacity planning; provisioned requires sizing read/write capacity units but can be cheaper at steady, predictable volume |
| **Path parameters (API Gateway)** | `{taskId}` in the resource path maps to `event['pathParameters']['taskId']` in the Lambda handler |
| **Least-privilege execution role** | Scope IAM actions to the specific resource ARN and specific actions the function needs — never attach a full-access managed policy to a Lambda role as a shortcut |

---

## Interview Prep: Common Questions

1. **"Why did you scope the IAM role to specific DynamoDB actions instead of using `AmazonDynamoDBFullAccess`?"** — Least privilege: if the function is compromised, the attacker can only put/get/query one table, not delete tables or read every table in the account.
2. **"What's a cold start and when does it matter?"** — The latency of provisioning a new execution environment; matters for latency-sensitive APIs with spiky/infrequent traffic, less so for steady high-volume traffic where environments stay warm.
3. **"When would you choose EC2/ASG over Lambda for an API?"** — Long-running requests beyond Lambda's execution limits, consistent high-volume traffic where reserved compute is cheaper, or workloads needing specialized runtimes/persistent local state Lambda doesn't support well.
4. **"How does this compare operationally to Lab 5?"** — No AMI/patch management, no capacity planning, no Multi-AZ configuration to think about — AWS handles all of that; the tradeoff is less control over performance consistency and cold-start behavior.

---

## Next Steps
- Add a `PUT`/`DELETE` method to complete full CRUD and practice IAM policy scoping for each new action
- Add an API Gateway usage plan with throttling and an API key to simulate rate-limiting a public API
- Continue to [Lab 7: Well-Architected Cost & Performance Review](aws-lab-7-well-architected-cost.md) to compare the actual cost of this lab's serverless model against Lab 5's always-on model at realistic traffic volumes
