# BLA8

# Three problems you can hit when working through the Governenace and QA of aws Gen AI Developer. Each one has the symptom, the cause, commands to diagnose it, the fix in code, and a way to check the fix worked.

# Lab Symptom	Root cause
CloudWatch Logs; Bedrock calls succeed but no invocation logs appear	Logging role trust or permissions, or the wrong Region.
CloudWatch Alarms; Alarm stays in INSUFFICIENT_DATA, and no email arrives	Wrong ModelId dimension and an unconfirmed SNS subscription.
AWS X-Ray;	Traces show Lambda but no Bedrock call, or no traces at all	Active tracing off, missing IAM permissions, or the SDK is not patched.

Before you start: replace ACCOUNT_ID, REGION (e.g. us-east-1) and the resource names with your own values. The examples use the AWS CLI v2 and Python 3.12 with boto3.

1. Bedrock model invocation logs never show up in CloudWatch

Symptom;
You turned on model invocation logging and the playground returns answers, but the log group is empty. The aws/bedrock/modelinvocations log stream never appears.

Root cause;
Bedrock writes logs by assuming an IAM role that you provide. Logs are silently dropped when any of these is true:

The role's trust policy does not allow bedrock.amazonaws.com to assume it.
The role has no logs:CreateLogStream / logs:PutLogEvents permission on the log group.
Logging was enabled in a different Region from the one you are invoking. The setting applies per account, per Region.
The log group does not exist yet.

Diagnose

# 1. Is logging configured in THIS Region, and with which role and log group?
aws bedrock get-model-invocation-logging-configuration --region REGION

# 2. Does the log group exist?
aws logs describe-log-groups --log-group-name-prefix /aws/bedrock/invocations --region REGION

# 3. Can Bedrock assume the role? Check the trust policy.
aws iam get-role --role-name BedrockInvocationLoggingRole \
  --query 'Role.AssumeRolePolicyDocument'


# Fix

Trust policy (trust.json). It allows only Bedrock in your account to assume the role:


{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "bedrock.amazonaws.com" },
    "Action": "sts:AssumeRole",
    "Condition": {
      "StringEquals": { "aws:SourceAccount": "ACCOUNT_ID" },
      "ArnLike": { "aws:SourceArn": "arn:aws:bedrock:REGION:ACCOUNT_ID:*" }
    }
  }]
}

Permissions policy (logs-policy.json):

json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["logs:CreateLogStream", "logs:PutLogEvents"],
    "Resource": "arn:aws:logs:REGION:ACCOUNT_ID:log-group:/aws/bedrock/invocations:log-stream:aws/bedrock/modelinvocations"
  }]
}

Create the log group and role, then enable logging:

bash
aws logs create-log-group --log-group-name /aws/bedrock/invocations --region REGION
aws logs put-retention-policy --log-group-name /aws/bedrock/invocations \
  --retention-in-days 30 --region REGION

aws iam create-role --role-name BedrockInvocationLoggingRole \
  --assume-role-policy-document file://trust.json
aws iam put-role-policy --role-name BedrockInvocationLoggingRole \
  --policy-name BedrockLogsWrite --policy-document file://logs-policy.json
python
# enable_invocation_logging.py
import boto3

REGION = "us-east-1"
ACCOUNT_ID = "123456789012"

bedrock = boto3.client("bedrock", region_name=REGION)

bedrock.put_model_invocation_logging_configuration(
    loggingConfig={
        "cloudWatchConfig": {
            "logGroupName": "/aws/bedrock/invocations",
            "roleArn": f"arn:aws:iam::{ACCOUNT_ID}:role/BedrockInvocationLoggingRole",
        },
        "textDataDeliveryEnabled": True,
        "imageDataDeliveryEnabled": False,
        "embeddingDataDeliveryEnabled": False,
    }
)
print(bedrock.get_model_invocation_logging_configuration()["loggingConfig"])

Large payloads (over 100 KB) and binary image data are delivered only to S3. If you log images, also add largeDataDeliveryS3Config with a bucket.

Verify
python
# verify_logging.py: make one call, then query the logs
import boto3, json, time

REGION = "us-east-1"
rt = boto3.client("bedrock-runtime", region_name=REGION)
rt.converse(
    modelId="anthropic.claude-3-haiku-20240307-v1:0",
    messages=[{"role": "user", "content": [{"text": "Say hello in five words."}]}],
)

time.sleep(60)  # delivery can take a minute
logs = boto3.client("logs", region_name=REGION)
q = logs.start_query(
    logGroupName="/aws/bedrock/invocations",
    startTime=int(time.time()) - 900,
    endTime=int(time.time()),
    queryString="fields @timestamp, modelId, input.inputTokenCount, output.outputTokenCount "
                "| sort @timestamp desc | limit 5",
)
time.sleep(5)
print(json.dumps(logs.get_query_results(queryId=q["queryId"])["results"], indent=2))

Exam takeaway: invocation logging is off by default, set per Region, and depends on a role that Bedrock assumes. CloudTrail does not capture prompt and response content. Only invocation logging does.

2. CloudWatch alarm on a Bedrock metric is stuck in INSUFFICIENT_DATA

Lab: CloudWatch Alarms – Hands On (Lecture 198)

Symptom

You created an alarm on InvocationLatency and sent plenty of traffic, but the alarm never leaves INSUFFICIENT_DATA. When you force it to ALARM, no email arrives.

Root cause

There are two independent problems:

Dimension mismatch. Bedrock publishes to the AWS/Bedrock namespace with a ModelId dimension. If the alarm's ModelId value differs from what was actually invoked, even by a version suffix or an inference-profile prefix such as us., the alarm watches a metric that has no data.
SNS subscription never confirmed. Email subscriptions remain PendingConfirmation until someone clicks the link, and SNS will not deliver to them.

A third point is related: Bedrock emits metrics only when there is traffic. With the default missing-data handling, a quiet model leaves the alarm in INSUFFICIENT_DATA.

Diagnose
bash
# Which ModelId values actually have data?
aws cloudwatch list-metrics --namespace AWS/Bedrock \
  --metric-name InvocationLatency --region REGION \
  --query 'Metrics[].Dimensions'

# Which dimensions is the alarm using?
aws cloudwatch describe-alarms --alarm-names bedrock-latency-high --region REGION \
  --query 'MetricAlarms[].{dims:Dimensions,state:StateValue,reason:StateReason}'

# Is the email subscription confirmed?
aws sns list-subscriptions-by-topic --region REGION \
  --topic-arn arn:aws:sns:REGION:ACCOUNT_ID:genai-alerts \
  --query 'Subscriptions[].{endpoint:Endpoint,arn:SubscriptionArn}'

If SubscriptionArn shows PendingConfirmation, the email was never confirmed. If the alarm's ModelId does not appear in list-metrics, the dimension is wrong.

Fix
python
# fix_bedrock_alarm.py
import boto3

REGION = "us-east-1"
ACCOUNT_ID = "123456789012"
TOPIC_ARN = f"arn:aws:sns:{REGION}:{ACCOUNT_ID}:genai-alerts"

cw = boto3.client("cloudwatch", region_name=REGION)

# 1. Use the ModelId that actually has data, not one typed by hand
dims = cw.list_metrics(Namespace="AWS/Bedrock", MetricName="InvocationLatency")["Metrics"]
model_ids = sorted({d["Value"] for m in dims for d in m["Dimensions"] if d["Name"] == "ModelId"})
print("ModelIds with data:", model_ids)
model_id = model_ids[0]

# 2. Recreate the alarm with the correct dimension and explicit missing-data handling
cw.put_metric_alarm(
    AlarmName="bedrock-latency-high",
    AlarmDescription="p90 Bedrock latency above 5s for 3 of 5 minutes",
    Namespace="AWS/Bedrock",
    MetricName="InvocationLatency",
    Dimensions=[{"Name": "ModelId", "Value": model_id}],
    ExtendedStatistic="p90",
    Period=60,
    EvaluationPeriods=5,
    DatapointsToAlarm=3,
    Threshold=5000,                  # milliseconds
    ComparisonOperator="GreaterThanThreshold",
    TreatMissingData="notBreaching", # no traffic means OK, not INSUFFICIENT_DATA
    AlarmActions=[TOPIC_ARN],
    OKActions=[TOPIC_ARN],
)
bash
# 3. Resubscribe and click the confirmation link in your inbox
aws sns subscribe --topic-arn arn:aws:sns:REGION:ACCOUNT_ID:genai-alerts \
  --protocol email --notification-endpoint you@example.com --region REGION

If the SNS topic is encrypted with a customer-managed KMS key, the key policy must also allow cloudwatch.amazonaws.com to use kms:Decrypt and kms:GenerateDataKey*. Otherwise CloudWatch cannot publish to the topic.

Verify
bash
# Force the alarm into ALARM and confirm the email arrives
aws cloudwatch set-alarm-state --alarm-name bedrock-latency-high \
  --state-value ALARM --state-reason "Manual test" --region REGION

# Check the alarm history to confirm the action ran
aws cloudwatch describe-alarm-history --alarm-name bedrock-latency-high \
  --history-item-type Action --max-records 3 --region REGION

Exam takeaway: an alarm is only as good as its dimensions. Use TreatMissingData for metrics that only appear with traffic. For token cost alarms, use the InputTokenCount and OutputTokenCount metrics with the Sum statistic.

3. X-Ray traces are missing, or show Lambda but no Bedrock call

Lab: AWS X-Ray – Hands On (Lecture 206)

Symptom

Either the X-Ray console shows no traces at all, or the trace shows the Lambda function as one solid bar with no Bedrock Runtime subsegment, so you cannot see how much of the latency the model call is responsible for.

Root cause
What you see	Why
No traces at all	Active tracing is off on the function or the API Gateway stage, or the execution role lacks xray:PutTraceSegments and xray:PutTelemetryRecords
Lambda appears but no Bedrock subsegment	The AWS SDK was not instrumented. patch_all() was never called, or it ran after the boto3 client was created
ModuleNotFoundError: aws_xray_sdk	The X-Ray SDK is not included in the Lambda runtime and must be packaged or added as a layer
Diagnose
bash
aws lambda get-function-configuration --function-name genai-chat \
  --query 'TracingConfig' --region REGION        

aws apigateway get-stage --rest-api-id API_ID --stage-name prod \
  --query 'tracingEnabled' --region REGION       
aws iam list-attached-role-policies --role-name genai-chat-role \
  --query 'AttachedPolicies[].PolicyName'           


Fix
Turn on tracing and grant permissions:


aws lambda update-function-configuration --function-name genai-chat \
  --tracing-config Mode=Active --region REGION

aws apigateway update-stage --rest-api-id API_ID --stage-name prod \
  --patch-operations op=replace,path=/tracingEnabled,value=true --region REGION

aws iam attach-role-policy --role-name genai-chat-role \
  --policy-arn arn:aws:iam::aws:policy/AWSXRayDaemonWriteAccess


Package the SDK:

mkdir -p package && pip install aws-xray-sdk -t package/
cp lambda_function.py package/ && (cd package && zip -r ../function.zip .)
aws lambda update-function-code --function-name genai-chat \
  --zip-file fileb://function.zip --region REGION

Instrument the handler. Patch the SDK before creating any client:


python
# lambda_function.py
import json
from aws_xray_sdk.core import xray_recorder, patch_all

patch_all()  # must run before boto3 clients are created

import boto3
bedrock = boto3.client("bedrock-runtime")  
MODEL_ID = "anthropic.claude-3-haiku-20240307-v1:0"

def lambda_handler(event, context):
    prompt = json.loads(event.get("body") or "{}").get("prompt", "Hello")

    # Custom subsegment with annotations you can filter on in the X-Ray console
    with xray_recorder.in_subsegment("generate_answer") as sub:
        sub.put_annotation("model_id", MODEL_ID)
        resp = bedrock.converse(
            modelId=MODEL_ID,
            messages=[{"role": "user", "content": [{"text": prompt}]}],
            inferenceConfig={"maxTokens": 300},
        )
        usage = resp["usage"]
        sub.put_annotation("output_tokens", usage["outputTokens"])
        sub.put_metadata("usage", usage)  # metadata is visible but not indexed

    answer = resp["output"]["message"]["content"][0]["text"]
    return {"statusCode": 200, "body": json.dumps({"answer": answer})}



