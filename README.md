# AWSGenAIdeveloper
# Section 2 — Generative AI Fundamentals and Amazon Bedrock

Errors encountered or anticipated during the labs, with the cause rather than the obvious-looking one. The pattern worth noticing: several Bedrock errors point at IAM when the real problem is model access, Region, or the wrong API client entirely.


## 1. `AccessDeniedException` on a model you have IAM permission for

```
An error occurred (AccessDeniedException) when calling the Converse operation:
You don't have access to the model with the specified model ID.
```

*Looks like:* an IAM policy problem.
*Usually is:* model access has not been granted for that model in that Region.

*Fix:* Bedrock console → *Model access* → request access → wait for
*Access granted*. Some third-party models require accepting the provider's EULA first.


## 2. *AccessDeniedException* naming a Region you never called

```
User: arn:aws:sts::...:assumed-role/AppRole/... is not authorized to perform:
bedrock:InvokeModel on resource: arn:aws:bedrock:us-west-2::foundation-model/...
```
…while your client is pinned to `us-east-1`.

*Cause:* you invoked through a cross-Region inference profile (`us.` prefix).
The request is authorized against the profile ARN and the underlying
foundation-model ARN in every Region the profile can route to.

*Fix:* grant the foundation-model ARN in all destination Regions. See the
`CrossRegionInferenceRequiresBothArns` statement in
`config/iam-bedrock-least-privilege.json`.


## 3. `ValidationException: The provided model identifier is invalid`

Three distinct causes, in the order worth checking:

1. **Typo or wrong version suffix.** The `:0` suffix is part of the ID.
2. **Wrong Region.** Model availability differs by Region; the ID is valid, the
   Region is not.
3. **The model is only reachable through an inference profile.** Some models
   have no on-demand base-ID entry point at all. Use the `us.`-prefixed
   profile ID as the `modelId`.

```bash
aws bedrock list-inference-profiles \
  --query 'inferenceProfileSummaries[].inferenceProfileId' --output text
```

## 4. `AttributeError: 'BedrockRuntime' object has no attribute 'converse'`

*Cause:* boto3 predates the Converse API.
*Fix:* `pip install --upgrade boto3 botocore` (>= 1.40 for the surface used
here). Worth checking early — the error names an attribute, not an SDK
version, so it reads like a typo.

## 5. `Invalid choice: 'retrieve'` from the AWS CLI

```
aws bedrock retrieve ...
Invalid choice: 'retrieve', maybe you meant: ...
```

**Cause:** wrong client. Bedrock is split across four:
**Fix:** `aws bedrock-agent-runtime retrieve ...`. This one looks like an
out-of-date CLI and is a mental-model problem.

## 6. `ThrottlingException: Too many requests`

*Cause:* account-level requests-per-minute or tokens-per-minute quota for
that model in that Region. Loops such as `04_inference_parameters_lab.py`
trigger it easily.

**Fixes, in order of preference:**

1. **Exponential backoff with jitter.** botocore has retries built in — raise
   the ceiling rather than writing your own:

   ```python
   from botocore.config import Config
   cfg = Config(retries={"max_attempts": 8, "mode": "adaptive"})
   client = boto3.client("bedrock-runtime", config=cfg)
   ```
2. **Cross-Region inference profile.** Spreading across Regions raises
   effective throughput — often the fastest real fix.
3. **Request a quota increase** in Service Quotas.
4. **Provisioned Throughput** for sustained high volume.

*Watch for the streaming variant:* on `ConverseStream`, throttling can arrive
as an in-band `throttlingException` **event** rather than a raised exception. A
loop that only handles `contentBlockDelta` will silently produce a truncated
answer with no error.

## 7. Knowledge Base returns nothing, or answers confidently from nothing

**Causes, in order of frequency:**

1. **The data source was never synced.** Creating a KB does not ingest
   anything. Run an ingestion job and confirm it reached `COMPLETE`.
2. **Ingestion partially failed.** Check
   `statistics.numberOfDocumentsFailed` — unsupported formats and
   oversized files are dropped individually and the job still reports success.
3. **Chunking mismatch.** Chunks too large bury the answer in noise; too small
   and they lose the context that makes them meaningful.
4. **Search type.** Queries containing codes or acronyms do poorly under pure
   `SEMANTIC`. Set `overrideSearchType: "HYBRID"`.

```bash
aws bedrock-agent list-ingestion-jobs --knowledge-base-id <KB> --data-source-id <DS> \
  --query 'ingestionJobSummaries[].[status,statistics.numberOfDocumentsScanned,statistics.numberOfDocumentsFailed]' \
  --output table
```

## 8. Guardrail does not fire

**Causes:**

 **Pointing at `DRAFT` vs a published version.** Edits land in `DRAFT`; a
  numbered version is frozen. Applications pinned to version `1` do not see
  your change.
 **Denied topic too narrowly defined.** Topic matching is semantic, and it
  leans heavily on the `examples` array. A definition with one example
  generalizes poorly — add three or four spanning different phrasings.
 **Filter strength too low.** `LOW`/`MEDIUM`/`HIGH` are real thresholds, not
  labels.
 **`PROMPT_ATTACK` configured with an output strength.** It is input-side
  only; the API rejects the configuration.
 **Contextual grounding without a cross-Region guardrail profile.** Grounding
  and automated-reasoning policies need `crossRegionConfig`.


## 9. `ValidationException` on the second turn of a tool-use loop

```
The toolResult block(s) at messages.1 do not have corresponding toolUse blocks
```

**Causes:**

 `toolUseId` in the `toolResult` does not exactly match the one issued.
 The assistant's `toolUse` message was not appended to `messages` before the
  `toolResult` message. The transcript must read: user → assistant(toolUse) →
  user(toolResult).
 The model requested **multiple** tools in one turn and only one result was
  returned. Every `toolUse` block needs a matching `toolResult` in the same
  message.


## 10. Embedding results look random

**Causes:**

**Mixing embedding models.** Vectors from different models are not
  comparable. Changing the embedding model means re-embedding the whole corpus.
- **Mismatched `dimensions`.** Query and corpus must use the same setting.
- **Comparing unnormalized vectors with a dot product.** Either set
  `normalize: true` or use true cosine similarity — not one and then the other.

---

## 12. Unexpected cost

**Where it actually comes from,** in descending order for a lab account:

1. **Vector store.** OpenSearch Serverless bills for OCUs whether or not you
   query it. A forgotten collection is the classic surprise bill from this
   section far more than the tokens.
2. **Provisioned Throughput.** Billed hourly from the moment it is created.
3. **Model choice.** The largest model can cost an order of magnitude more per
   token than a small one for a task the small one handles.
4. **Output tokens.** Priced well above input. Verbose system prompts asking
   for "detailed explanations" are a recurring line item.
5. **Invocation logs.** Full prompt/completion capture in S3 and CloudWatch
   accumulates quickly under load.

**Controls:** AWS Budgets with an alert, cost-allocation tags on every
resource, `requestMetadata` on Converse calls for per-application attribution,
and the `Invocations` / `InputTokenCount` CloudWatch metrics on a dashboard.

