# X-Ray Trace Missing the Bedrock Call

## Problem

The X-Ray trace shows the Lambda function, but the Amazon Bedrock call does not appear as a subsegment. The boto3 client was created before the AWS SDK was patched for X-Ray.

## Fix

```python
import json
from aws_xray_sdk.core import patch_all

patch_all()

import boto3

bedrock = boto3.client("bedrock-runtime")


def lambda_handler(event, context):
    prompt = json.loads(event.get("body") or "{}").get("prompt", "Hello")
    resp = bedrock.converse(
        modelId="anthropic.claude-3-haiku-20240307-v1:0",
        messages=[{"role": "user", "content": [{"text": prompt}]}],
    )
    answer = resp["output"]["message"]["content"][0]["text"]
    return {"statusCode": 200, "body": json.dumps({"answer": answer})}
```


