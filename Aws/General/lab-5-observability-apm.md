# Lab 5: Observability & Application Performance Monitoring

Check box if done: [ ]

## Overview
"It's slow sometimes" is not an actionable bug report, and neither is a wall of unstructured CloudWatch logs with no way to correlate a single request across services. This lab builds the observability stack that turns "slow sometimes" into "the third-party API call in this one Lambda path adds 800ms at p99, and here's the trace that proves it": X-Ray distributed tracing, CloudWatch Logs Insights for structured queries instead of `grep`, a dashboard that answers "is this healthy" at a glance, and composite alarms that model an actual SLO instead of alerting on raw CPU.

**Estimated time**: 60–75 minutes
**Cost**: ~$0–$1 (Lambda, API Gateway HTTP API, and the first 100,000 X-Ray traces/month are all within AWS's free tier at this lab's scale; CloudWatch Dashboards bill $3/month per dashboard past the first 3 — stay under that limit and this section is free too)

---

## Scenario
A Lambda-backed API in your org has an intermittent latency complaint with no supporting evidence — nobody can say which part of the request is slow, whether it's the function itself or a downstream call, or how often it actually happens versus how often someone just remembers a bad experience. You're instrumenting it properly: X-Ray tracing showing exactly where time goes inside a single request, Logs Insights queries that answer "how often does this actually happen" in seconds instead of eyeballing raw logs, a dashboard for at-a-glance health, and a composite alarm that fires only when both error rate and latency are simultaneously bad — an actual SLO burn signal, not noise from one spiky metric.

---

## Objectives
- Deploy a small instrumented API (Lambda + API Gateway) with X-Ray active tracing enabled
- Read a distributed trace end-to-end, including a downstream subsegment
- Write and run CloudWatch Logs Insights queries against structured Lambda logs
- Build a CloudWatch dashboard combining metric and log-based widgets
- Configure an SLO-style composite alarm combining error rate and latency

---

## Part 1: Deploy an Instrumented API

### Step 1: Write the Lambda Function
This function makes a real downstream call (to S3, standing in for "a database or another service") so the trace in Part 2 has more than one segment to show.

```python
# app.py
import json
import time
import boto3
from aws_xray_sdk.core import xray_recorder, patch_all

patch_all()  # instruments boto3 calls automatically for X-Ray

s3 = boto3.client('s3')

def handler(event, context):
    with xray_recorder.in_subsegment('list_demo_bucket'):
        s3.list_buckets()
        time.sleep(0.1)  # simulate realistic downstream latency

    return {
        'statusCode': 200,
        'headers': {'Content-Type': 'application/json'},
        'body': json.dumps({'message': 'ok', 'timestamp': time.time()}),
    }
```
`patch_all()` from the X-Ray SDK instruments `boto3` calls automatically — the `s3.list_buckets()` call becomes its own subsegment in the trace with no manual timing code needed. The explicit `in_subsegment` block adds a named segment around it so it's identifiable in the trace timeline rather than lumped into the handler's total duration.

### Step 2: Package and Deploy With Active Tracing
```bash
pip install aws-xray-sdk -t package/
cp app.py package/
cd package && zip -r ../function.zip . && cd ..

aws iam create-role \
  --role-name lab5-lambda-role \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{"Effect": "Allow", "Principal": {"Service": "lambda.amazonaws.com"}, "Action": "sts:AssumeRole"}]
  }'

aws iam attach-role-policy --role-name lab5-lambda-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam attach-role-policy --role-name lab5-lambda-role \
  --policy-arn arn:aws:iam::aws:policy/AWSXRayDaemonWriteAccess
aws iam attach-role-policy --role-name lab5-lambda-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

aws lambda create-function \
  --function-name lab5-demo-api \
  --runtime python3.12 \
  --handler app.handler \
  --role arn:aws:iam::<AWS_ACCOUNT_ID>:role/lab5-lambda-role \
  --zip-file fileb://function.zip \
  --tracing-config Mode=Active \
  --timeout 10
```
`--tracing-config Mode=Active` is the whole switch — Lambda automatically sends trace segments to X-Ray for every invocation once this is set, no separate agent to deploy the way EC2/ECS tracing requires the X-Ray daemon running as a sidecar.

### Step 3: Front It With an HTTP API
```bash
aws apigatewayv2 create-api \
  --name lab5-demo-api-gw \
  --protocol-type HTTP \
  --target arn:aws:lambda:us-east-1:<AWS_ACCOUNT_ID>:function:lab5-demo-api

API_ID=$(aws apigatewayv2 get-apis --query "Items[?Name=='lab5-demo-api-gw'].ApiId" --output text)

aws lambda add-permission \
  --function-name lab5-demo-api \
  --statement-id apigw-invoke \
  --action lambda:InvokeFunction \
  --principal apigateway.amazonaws.com \
  --source-arn "arn:aws:execute-api:us-east-1:<AWS_ACCOUNT_ID>:$API_ID/*/*"
```
`--target` with `create-api` provisions a quick-create HTTP API with a default route and integration — the fastest path to a working endpoint without hand-defining routes and integrations separately.

**Validation checkpoint**:
```bash
API_ENDPOINT=$(aws apigatewayv2 get-apis --query "Items[?Name=='lab5-demo-api-gw'].ApiEndpoint" --output text)
curl $API_ENDPOINT
```
Expect `{"message": "ok", "timestamp": ...}`.

---

## Part 2: Read a Distributed Trace

### Step 4: Generate Traffic
```bash
for i in $(seq 1 20); do curl -s $API_ENDPOINT > /dev/null; done
```

### Step 5: Pull Trace Summaries
```bash
aws xray get-trace-summaries \
  --start-time $(date -d '-10 minutes' +%s) \
  --end-time $(date +%s) \
  --query "TraceSummaries[*].{Id:Id,Duration:Duration,ResponseTime:ResponseTime}"
```

### Step 6: Pull a Full Trace and Inspect Segments
```bash
TRACE_ID=$(aws xray get-trace-summaries \
  --start-time $(date -d '-10 minutes' +%s) --end-time $(date +%s) \
  --query "TraceSummaries[0].Id" --output text)

aws xray batch-get-traces --trace-ids $TRACE_ID \
  --query "Traces[0].Segments[*].Document" --output text
```
**Reading this**: the trace document contains the Lambda invocation as the top-level segment, with the `list_demo_bucket` subsegment from Step 1 nested inside it, each carrying its own start/end timestamps. This is what makes distributed tracing different from a log line with a duration in it — you can see *where inside* the request the time actually went, not just the total.

**Validation checkpoint**: the subsegment's duration is roughly 100ms (matching the `time.sleep(0.1)` in Step 1), and it's clearly a smaller slice nested inside the total invocation duration — proof the instrumentation is capturing real sub-request timing, not just wrapping the whole function in one opaque span.

---

## Part 3: CloudWatch Logs Insights

### Step 7: Query Recent Invocations
```bash
QUERY_ID=$(aws logs start-query \
  --log-group-name /aws/lambda/lab5-demo-api \
  --start-time $(date -d '-1 hour' +%s) \
  --end-time $(date +%s) \
  --query-string 'fields @timestamp, @duration, @requestId | sort @timestamp desc | limit 20' \
  --query "queryId" --output text)

sleep 5
aws logs get-query-results --query-id $QUERY_ID
```

### Step 8: Query for Errors Specifically
```bash
aws logs start-query \
  --log-group-name /aws/lambda/lab5-demo-api \
  --start-time $(date -d '-1 hour' +%s) \
  --end-time $(date +%s) \
  --query-string 'fields @timestamp, @message | filter @message like /ERROR/ | sort @timestamp desc | limit 20'
```
**Why this beats scrolling raw logs**: Logs Insights runs a purpose-built query language directly against CloudWatch Logs without needing to export to a separate analytics store first — `filter`, `stats`, and `parse` cover the majority of "how often does X happen" questions that would otherwise mean piping raw log output through `grep`/`awk` by hand.

---

## Part 4: Build a CloudWatch Dashboard

### Step 9: Create It
```bash
cat > dashboard.json << 'EOF'
{
  "widgets": [
    {
      "type": "metric",
      "x": 0, "y": 0, "width": 12, "height": 6,
      "properties": {
        "title": "Invocations, Errors, Duration",
        "metrics": [
          ["AWS/Lambda", "Invocations", "FunctionName", "lab5-demo-api", {"stat": "Sum"}],
          ["AWS/Lambda", "Errors", "FunctionName", "lab5-demo-api", {"stat": "Sum"}],
          ["AWS/Lambda", "Duration", "FunctionName", "lab5-demo-api", {"stat": "p99"}]
        ],
        "period": 300,
        "region": "us-east-1"
      }
    },
    {
      "type": "log",
      "x": 0, "y": 6, "width": 12, "height": 6,
      "properties": {
        "title": "Recent Errors",
        "query": "SOURCE '/aws/lambda/lab5-demo-api' | fields @timestamp, @message | filter @message like /ERROR/ | sort @timestamp desc | limit 20",
        "region": "us-east-1",
        "view": "table"
      }
    }
  ]
}
EOF

aws cloudwatch put-dashboard \
  --dashboard-name lab5-demo-api-health \
  --dashboard-body file://dashboard.json
```
A log-type widget embeds a Logs Insights query directly on the dashboard — Step 8's query and this widget are the same underlying mechanism, just one runs ad hoc and the other renders continuously.

**Validation checkpoint**: `aws cloudwatch get-dashboard --dashboard-name lab5-demo-api-health` returns the dashboard body unchanged, and it's visible in the CloudWatch console with both widgets rendering data from Step 4's traffic.

---

## Part 5: SLO-Style Composite Alarms

### Step 10: Why Not Just Alarm on One Metric
A single-metric alarm on error rate fires on a single bad minute even if latency is fine, and a single-metric alarm on latency fires on one slow request even if the error rate is zero — either alone is noisy enough that teams learn to ignore it. A **composite alarm** models an actual SLO breach: both conditions bad at once, sustained, is a real signal worth paging on.

### Step 11: Create the Two Child Alarms
```bash
aws cloudwatch put-metric-alarm \
  --alarm-name lab5-high-error-rate \
  --namespace AWS/Lambda \
  --metric-name Errors \
  --dimensions Name=FunctionName,Value=lab5-demo-api \
  --statistic Sum \
  --period 300 \
  --evaluation-periods 1 \
  --threshold 3 \
  --comparison-operator GreaterThanOrEqualToThreshold

aws cloudwatch put-metric-alarm \
  --alarm-name lab5-high-p99-latency \
  --namespace AWS/Lambda \
  --metric-name Duration \
  --dimensions Name=FunctionName,Value=lab5-demo-api \
  --extended-statistic p99 \
  --period 300 \
  --evaluation-periods 1 \
  --threshold 2000 \
  --comparison-operator GreaterThanThreshold
```

### Step 12: Create the Composite Alarm and an SNS Topic to Notify
```bash
aws sns create-topic --name lab5-slo-alerts
TOPIC_ARN=$(aws sns list-topics --query "Topics[?ends_with(TopicArn, ':lab5-slo-alerts')].TopicArn" --output text)
aws sns subscribe --topic-arn $TOPIC_ARN --protocol email --notification-endpoint <your-email>

aws cloudwatch put-composite-alarm \
  --alarm-name lab5-slo-breach \
  --alarm-rule "ALARM(lab5-high-error-rate) AND ALARM(lab5-high-p99-latency)" \
  --alarm-actions $TOPIC_ARN
```
`--alarm-rule` is a boolean expression over other alarms' states — `AND` here means the composite only fires when both child alarms are simultaneously in `ALARM`, which is the SLO-burn signal from Step 10, not either symptom alone.

**Validation checkpoint**:
```bash
aws cloudwatch describe-alarms --alarm-names lab5-slo-breach --query "CompositeAlarms[0].StateValue"
```
Expect `OK` under normal traffic (both child alarms healthy). Confirm the SNS subscription with the confirmation email before relying on this in a real scenario — an unconfirmed subscription silently delivers nothing.

---

## Cleanup

```bash
# Alarms
aws cloudwatch delete-alarms --alarm-names lab5-slo-breach lab5-high-error-rate lab5-high-p99-latency

# SNS topic
aws sns delete-topic --topic-arn $TOPIC_ARN

# Dashboard
aws cloudwatch delete-dashboards --dashboard-names lab5-demo-api-health

# API Gateway
aws apigatewayv2 delete-api --api-id $API_ID

# Lambda and its role
aws lambda delete-function --function-name lab5-demo-api
aws iam detach-role-policy --role-name lab5-lambda-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam detach-role-policy --role-name lab5-lambda-role --policy-arn arn:aws:iam::aws:policy/AWSXRayDaemonWriteAccess
aws iam detach-role-policy --role-name lab5-lambda-role --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
aws iam delete-role --role-name lab5-lambda-role

# Log group (Lambda creates this automatically; delete it explicitly, it doesn't disappear with the function)
aws logs delete-log-group --log-group-name /aws/lambda/lab5-demo-api
```

---

## Key Concepts

| Term | Definition |
|------|------------|
| **X-Ray trace** | The full timeline of a single request across every instrumented service it touched, made of one or more segments/subsegments — what makes "where did the time go" answerable instead of guessed |
| **Segment vs. subsegment** | A segment is the top-level unit of work (a Lambda invocation, a request handled by an EC2 service); a subsegment is a nested, named slice of that work (a specific downstream call) |
| **`patch_all()`** | The X-Ray SDK call that auto-instruments supported libraries (`boto3` included) so downstream AWS API calls appear as subsegments without manual timing code |
| **CloudWatch Logs Insights** | A purpose-built query language run directly against CloudWatch Logs — `filter`/`stats`/`parse` answer frequency and pattern questions without exporting logs to a separate store first |
| **Composite alarm** | A CloudWatch alarm whose state is a boolean expression over other alarms' states, not a metric threshold itself — the mechanism for modeling "both conditions bad at once" instead of alerting on one noisy metric |
| **SLO-style alerting** | Alerting on a sustained, compound condition that reflects real user impact (elevated errors *and* elevated latency together) rather than any single metric crossing a threshold in isolation |

---

## Common Mistakes
- **Alerting on a single raw metric (CPU, one error) instead of a compound condition**: high noise, low signal — teams learn to ignore alarms that fire on normal variance
- **Forgetting the Lambda execution role needs `AWSXRayDaemonWriteAccess` in addition to basic execution**: without it, `--tracing-config Mode=Active` is set but traces silently never appear, and it looks like a code bug instead of a permissions gap
- **Not confirming the SNS email subscription**: an unconfirmed subscription delivers nothing, and the composite alarm will fire correctly while nobody actually gets notified
- **Creating more than 3 CloudWatch dashboards without checking cost**: the first 3 are free; every dashboard beyond that bills $3/month regardless of how often it's viewed
- **Treating a single slow trace as proof of a systemic issue**: one trace shows what happened in one request — use Logs Insights (Part 3) or X-Ray's trace-group aggregate view to confirm a pattern before treating an outlier as the norm

---

## Next Steps
This lab builds on the Lambda + API Gateway pattern from [SAA-C03 Lab 6: Serverless Architecture](../SAA-C03/aws-lab-6-serverless.md) — same building blocks, this time watching what the deployed app does instead of just deploying it. For the security-monitoring side of CloudWatch/CloudTrail (detecting *who did what* rather than *how the app performed*), see [SCS-C02 Lab 2: Detection & Monitoring](../SCS-C02/aws-sec-lab-2-detection-monitoring.md) — genuinely different questions, both answered by overlapping tooling. [Lab 6](lab-6-cost-management-finops.md) continues with a related instinct: this lab makes performance measurable, Lab 6 makes spend measurable.
