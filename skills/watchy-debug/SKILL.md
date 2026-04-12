---
name: watchy-debug
description: >-
  Investigate AWS Lambda production errors using Watchy's operational graph,
  CloudWatch logs, git history, and CloudTrail events. Follows a senior
  debugger's playbook to find root cause and suggest fixes.
  Triggers: "debug this error", "investigate 500", "what's wrong with my Lambda",
  "production error", "why is this failing", "investigate errors"
argument-hint: "[error description, error group ID, or Lambda function name]"
disable-model-invocation: false
user-invocable: true
---

# Watchy Error Investigation Playbook

You are an expert AWS Lambda debugger. Follow this playbook precisely. Each step tells you exactly what to run, what field to extract, and what to do next.

## Tools You Will Use

- **Bash** — curl (Watchy API), aws CLI (CloudWatch, CloudWatch Metrics, SQS), git, find/grep
- **Read** — to open source files identified from stack traces
- **Grep** — to find files related to a Lambda function name

## Prerequisites Check

Before starting, verify you have what you need. Run these in parallel:

```bash
echo "WATCHY_API_KEY: ${WATCHY_API_KEY:+set}"
```
```bash
command -v aws >/dev/null 2>&1 && echo "aws: installed" || echo "aws: missing"
```
```bash
aws sts get-caller-identity --query Account --output text 2>&1 | head -1
```

**If `WATCHY_API_KEY` is not set:** Ask the developer for it. They can create one at watchy.dev -> Settings -> API Keys (needs `read:errors` + `read:stacks` scopes). Stop here until they provide it.

### AWS CLI not installed

**If `aws` is missing**, offer to install it. Detect the OS:

```bash
uname -s
```

- **Darwin (macOS)**: `brew install awscli` (or download the pkg from https://awscli.amazonaws.com/AWSCLIV2.pkg)
- **Linux**:
  ```bash
  curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "/tmp/awscliv2.zip" && unzip -q /tmp/awscliv2.zip -d /tmp && sudo /tmp/aws/install
  ```
- **Windows**: download the MSI installer from https://awscli.amazonaws.com/AWSCLIV2.msi

Ask the developer which they prefer, then run the install command (or have them run it). Verify with `aws --version`. Proceed to the credential step below.

### Multi-account reality

Developers typically manage multiple AWS accounts (dev, staging, prod, multiple clients). The error you're investigating runs in a **specific account** (you'll get it from Step 1 as `data.error.group.awsAccountId` — call this `ERROR_ACCOUNT_ID`). CloudWatch calls must be made against that account, not whatever the default profile is.

**If `aws sts` succeeds:** Store the authenticated account as `AWS_ACCOUNT`. You'll validate this matches `ERROR_ACCOUNT_ID` after Step 1. If it doesn't match, see the "Switch to the right account" section below.

**First, check which profiles they already have configured.** Many devs already have profiles for each account — just pick the right one instead of setting up new creds.

```bash
aws configure list-profiles 2>/dev/null
```

If profiles exist, after Step 1 runs and you know `ERROR_ACCOUNT_ID`, check each one:

```bash
for p in $(aws configure list-profiles); do
  acct=$(aws sts get-caller-identity --profile "$p" --query Account --output text 2>/dev/null)
  echo "$p -> $acct"
done
```

Match `ERROR_ACCOUNT_ID` against the output. If found, prepend `AWS_PROFILE=MATCHED_PROFILE` to all subsequent `aws` calls.

### AWS CLI installed but no matching profile (or credentials expired)

**If no profile matches `ERROR_ACCOUNT_ID`, or `aws sts` fails entirely, or returns `ExpiredToken`:** Don't silently skip. Ask the developer:

> "I need AWS credentials for account `ERROR_ACCOUNT_ID` to pull CloudWatch logs. How do you want to authenticate?
>
> 1. **Paste temporary credentials** — fastest, session-only. If you use AWS SSO, grab the three `export` lines from your access portal for account `ERROR_ACCOUNT_ID` (Option 1: Set AWS environment variables). Paste them here.
> 2. **Add an SSO profile for this account** — recommended if you'll investigate errors in this account regularly. I'll set up a profile scoped to account `ERROR_ACCOUNT_ID`.
> 3. **Use an existing profile** — if you know you have a profile for this account but it wasn't auto-detected, tell me the profile name.
> 4. **Configure long-lived IAM keys** — via `aws configure`. Only if SSO isn't available.
> 5. **Skip CloudWatch** — continue with Watchy API + git only.
>
> Which do you prefer? (1/2/3/4/5)"

**If the developer chooses 1 (paste creds)**: they will appear in your next user message as `export AWS_ACCESS_KEY_ID=...`, `export AWS_SECRET_ACCESS_KEY=...`, `export AWS_SESSION_TOKEN=...`. Prepend all three `export` commands to every subsequent `aws` call in this session. Re-run `aws sts get-caller-identity` to verify the account matches `ERROR_ACCOUNT_ID`. If it doesn't match, the dev grabbed creds for the wrong account — ask again.

**If the developer chooses 2 (SSO profile)**: Ask for a profile name that identifies the account (e.g. `myclient-prod`, `acme-dev`). Run:
```bash
aws configure sso --profile PROFILE_NAME
```
They'll provide SSO start URL, region, then pick account `ERROR_ACCOUNT_ID` + role from the browser. Verify with `aws sts get-caller-identity --profile PROFILE_NAME --query Account --output text` — must equal `ERROR_ACCOUNT_ID`. Prepend `AWS_PROFILE=PROFILE_NAME` to all subsequent `aws` calls.

**If the developer chooses 3 (existing profile)**: They'll give you a profile name. Verify it maps to `ERROR_ACCOUNT_ID`:
```bash
aws sts get-caller-identity --profile PROFILE_NAME --query Account --output text
```
If it matches, prepend `AWS_PROFILE=PROFILE_NAME` to all subsequent `aws` calls. If not, tell the dev and re-ask.

**If the developer chooses 4 (IAM keys)**: Run `aws configure --profile PROFILE_NAME` (use a scoped profile name, not default — they have multiple accounts). Default region should be the error's region from Step 1. Verify the account matches. Prepend `AWS_PROFILE=PROFILE_NAME` to subsequent calls.

**If the developer chooses 5 (skip)**: Note `AWS_CLI_UNAVAILABLE = true` and `CLOUDWATCH_SKIPPED = true`. Continue without CloudWatch.

### Switch to the right account (when default profile is wrong account)

If `aws sts` succeeds with a different account than `ERROR_ACCOUNT_ID`, don't make AWS calls yet. Run the profile-matching loop above to find a profile for `ERROR_ACCOUNT_ID`. If found, use `AWS_PROFILE=MATCHED_PROFILE` for all calls. If not found, ask the developer via the 5-option prompt above.

### Resolve Base URL

```bash
curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/api/v1/stacks 2>/dev/null
```

- If this returns `401` or `200` → use `http://localhost:3000`
- Otherwise → use `https://watchy.dev`

Store the result mentally as `BASE_URL` for all subsequent API calls.

---

## Route: Pick Your Entry Point

**The developer provided an error group ID** (starts with `eg-` or is a UUID, or was in the copied prompt):
→ Go to **Step 1** with that ID.

**The developer described an error** (function name, error message, endpoint, or status code):
→ Go to **Step 0** to find it in Watchy.

**The developer just said "check for errors" with no specifics:**
→ Go to **Step 0** with no filters.

---

## Step 0: Find the Error

Run this to search for recent unresolved errors:

```bash
curl -s -H "Authorization: Bearer $WATCHY_API_KEY" \
  "$BASE_URL/api/v1/errors?time_range=24h&status=unresolved&include_traces=true&limit=20"
```

**Parse the response:**
- `data.groups` — array of error groups
- Each group has: `id`, `errorName`, `errorMessage`, `functionName`, `eventCount`, `criticality`

**If the developer mentioned a function name**, scan `functionName` fields to find a match.
**If the developer mentioned an error type**, scan `errorName` fields.
**If multiple matches**, pick the one with highest `criticality` or highest `eventCount`.

**If `data.groups` is empty or the API call fails:**
The Lambda may not have Watchy error capture installed. Go directly to CloudWatch:

```bash
aws logs filter-log-events \
  --log-group-name "/aws/lambda/FUNCTION_NAME" \
  --filter-pattern "ERROR" \
  --start-time $(date -d '24 hours ago' +%s000 2>/dev/null || date -v-24H +%s000) \
  --region REGION \
  --limit 20
```

Replace `FUNCTION_NAME` with what the developer mentioned. If you don't know the region, try `us-east-1` or ask.

If you had to use CloudWatch directly, set a mental flag: `WATCHY_CAPTURE_NOT_INSTALLED = true`. Continue to Step 2 with what you found. Skip Step 1.

**If you found the error in Watchy**, extract the `id` field from the matching group. Proceed to Step 1.

---

## Step 1: Get Full Investigation Context

```bash
curl -s -H "Authorization: Bearer $WATCHY_API_KEY" \
  "$BASE_URL/api/v1/errors/ERROR_GROUP_ID/investigate"
```

Replace `ERROR_GROUP_ID` with the ID from Step 0 or from the developer's prompt.

**Parse the response and extract these fields — you will use them in subsequent steps:**

| What you need | JSON path | Used in |
|---|---|---|
| Function name | `data.resource.name` (fallback: `data.error.group.functionName`) | Step 4 (git search) |
| Log group | `data.resource.config.logGroup` | Step 2, Step 3 |
| Request ID | `data.error.recentEvents[0].requestId` | Step 2, Step 3 |
| Region | `data.error.group.awsRegion` | Step 2, Step 3, Step 5 |
| Memory limit | `data.resource.config.memorySize` | Step 3 (OOM check) |
| Timeout | `data.resource.config.timeout` | Step 3 (timeout check) |
| Stack trace | `data.error.recentEvents[0].stackTrace` | Step 2 (read source) |
| Cold start | `data.error.recentEvents[0].coldStart` | Step 3 (init duration) |
| Business context | `data.error.recentEvents[0].context` | Step 6 (root cause) |
| Upstream resources | `data.graph.upstream[]` — each has `name`, `type`, `edgeType` | Step 6 (what triggers this) |
| Downstream resources | `data.graph.downstream[]` — each has `name`, `type`, `edgeType` | Step 5 (dependency check) |
| Recent deployments | `data.recentDeployments[]` — each has `eventName`, `eventTime`, `userName` | Step 4 (correlate) |
| Error trend | `data.history.trend` — `"increasing"`, `"decreasing"`, or `"stable"` | Step 6 (severity) |
| Occurrence count | `data.error.group.eventCount` | Step 6 |
| Criticality | `data.error.group.criticality` | Step 6 |

**If the API returns an error:**
- `401` → API key is invalid or missing scope. Ask the developer.
- `404` → Error group ID is wrong. Go back to Step 0.
- Other error → Log it, proceed with whatever data the developer provided in the prompt. Skip to Step 2.

Now read the stack trace. Identify the source file and line number from the top frame. Use **Read** to open that file if it exists in the local codebase. The path in the stack trace typically maps to a local path like `src/services/order.service.ts` (strip `/var/task/` prefix, change `.js` to `.ts` if the project uses TypeScript).

---

## Step 2: Pull CloudWatch Logs

**Requires:** `logGroup` and `requestId` from Step 1. Also requires `AWS_CLI_UNAVAILABLE` to NOT be set.

**If `AWS_CLI_UNAVAILABLE` is set**, tell the developer:
> "Skipping CloudWatch logs — AWS CLI is not configured. CloudWatch logs show what happened before the crash (DB failures, permission errors, input data) and whether this is an OOM/timeout. Want to configure AWS credentials and retry?"

If the developer says no, continue to Step 4. Note `CLOUDWATCH_SKIPPED = true` in your analysis.

**If `logGroup` or `requestId` is missing**, tell the developer:
> "Skipping CloudWatch logs — resource not yet linked in Watchy (log group unknown). The resource sync job runs every 15 minutes. You can also provide the log group manually if you know it (format: `/aws/lambda/FUNCTION_NAME`)."

If the developer provides a log group, use it. Otherwise continue to Step 4 with `CLOUDWATCH_SKIPPED = true`.

### Account Validation

**Before making any AWS call**, verify the authenticated AWS account matches the error's account. Run:

```bash
aws sts get-caller-identity --query Account --output text
```

(Prepend `AWS_PROFILE=PROFILE_NAME` if a profile was selected in prerequisites, or the three `export` commands if temp creds were pasted.)

Compare the output to `ERROR_ACCOUNT_ID` (from Step 1's `data.error.group.awsAccountId`).

**If they don't match**, go back to the "Switch to the right account" section in Prerequisites. Run the profile-matching loop to find a profile for `ERROR_ACCOUNT_ID`, or ask the developer to pick from the 5 authentication options. Do NOT make AWS calls against the wrong account — the log group won't exist there and the error message will be confusing.

**Security note**: Temporary credentials (session tokens) expire in 1-12 hours. If a later AWS call fails with `ExpiredToken`, ask for fresh credentials — don't retry with the expired ones.

### Pull Logs

```bash
aws logs filter-log-events \
  --log-group-name "LOG_GROUP" \
  --filter-pattern "REQUEST_ID" \
  --region REGION
```

Replace `LOG_GROUP`, `REQUEST_ID`, `REGION` with the values extracted in Step 1.

**If this fails with `AccessDeniedException`:** The developer's AWS credentials don't have CloudWatch access. Tell the developer explicitly — do NOT silently skip. Ask:
> "CloudWatch access denied. Your IAM role/user needs `logs:FilterLogEvents` permission. Want to skip CloudWatch and continue, or fix permissions first?"

**If this fails with `ResourceNotFoundException`:** The log group doesn't exist — the Lambda may have been deleted or renamed. Note this and skip to Step 4.

**What to look for in the output:**
- Error messages or exceptions logged before the crash
- Database query failures (`ECONNREFUSED`, `Timeout`, `ThrottlingException`)
- HTTP call failures (status codes, timeouts)
- Permission errors (`AccessDenied`, `is not authorized`)
- The actual input/event that caused the error

---

## Step 3: Check Lambda Execution Metadata

**Requires:** `logGroup` and `requestId` from Step 1. Skip if `CLOUDWATCH_SKIPPED` is set.

```bash
aws logs filter-log-events \
  --log-group-name "LOG_GROUP" \
  --filter-pattern '"REPORT" "REQUEST_ID"' \
  --region REGION
```

**Note:** The filter pattern must use double quotes inside single quotes to match literal strings. The `:` character in CloudWatch filter patterns needs quoting.

The REPORT line looks like:
```
REPORT RequestId: xxx Duration: 1500.00 ms Billed Duration: 1500 ms Memory Size: 256 MB Max Memory Used: 240 MB Init Duration: 800.00 ms
```

**Decision tree:**

| Condition | Diagnosis | Action |
|---|---|---|
| `Max Memory Used` > 90% of `Memory Size` (from Step 1) | OOM — out of memory | Suggest increasing memory or optimizing allocations |
| `Duration` > 80% of `Timeout` (from Step 1, in seconds × 1000) | Timeout risk — slow execution | Suggest increasing timeout or finding the slow operation in Step 2 logs |
| `Init Duration` present AND > 5000 ms | Slow cold start | Suggest lazy-loading SDKs, reducing bundle size |
| `Init Duration` present AND `coldStart` is true (from Step 1) | Error during cold start | Initialization code is likely the culprit — check DB connection setup, config loading |
| None of the above | Resource usage is normal | The error is in application logic, not infrastructure |

---

## Step 4: Check Git History

**Requires:** function name from Step 1.

First, find files related to this Lambda:

```bash
grep -rl "FUNCTION_NAME" --include="*.ts" --include="*.js" -l . 2>/dev/null | head -10
```

Replace `FUNCTION_NAME` with the function name. If the function name is long (e.g. `stratos-prod-orders-create-order`), try the last meaningful segment (e.g. `create-order` or `createOrder`).

If that returns no results, try the handler file path from the stack trace:
```bash
git log --since="48 hours ago" --oneline -- "src/services/order.service.ts" "src/handlers/webhook.handler.ts"
```

(Use the actual file paths from the stack trace, stripped of `/var/task/` prefix.)

**If there were recent commits:**
```bash
git show COMMIT_HASH --stat
git diff COMMIT_HASH~1 COMMIT_HASH -- MATCHING_FILES
```

**Correlate with deployments:** Compare commit timestamps against `recentDeployments` from Step 1. If a deployment (`UpdateFunctionCode` or `UpdateFunctionConfiguration`) happened within 1 hour of the error's `firstSeenAt`, that deploy is the likely trigger.

---

## Step 5: Check Downstream Dependencies

**Requires:** `data.graph.downstream` from Step 1.

If `downstream` is empty, skip this step.

For each downstream resource, run the appropriate check based on its `type` field:

### type: `dynamodb_table`
```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/DynamoDB \
  --metric-name ThrottledRequests \
  --dimensions Name=TableName,Value=TABLE_NAME \
  --start-time $(date -u -v-1H +%Y-%m-%dT%H:%M:%S 2>/dev/null || date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 --statistics Sum --region REGION
```
Replace `TABLE_NAME` with the resource `name` from the graph. If Sum > 0, the table is throttling.

### type: `sqs_queue`
```bash
aws sqs get-queue-attributes \
  --queue-url "https://sqs.REGION.amazonaws.com/ACCOUNT_ID/QUEUE_NAME" \
  --attribute-names ApproximateNumberOfMessages ApproximateNumberOfMessagesNotVisible \
  --region REGION
```
Replace `QUEUE_NAME` with the resource `name`. If message count is very high, the queue is backed up.

### type: `s3_bucket`
Check if the Lambda role has access. Look at the CloudWatch logs from Step 2 for `AccessDenied` errors mentioning S3.

### type: `sns_topic`
Check CloudWatch logs from Step 2 for publish failures.

### Any other type
Note it in the analysis but don't run a specific check. The CloudWatch logs from Step 2 likely already show the failure.

---

## Step 6: Produce Root Cause Analysis

Gather everything you've learned and present this structured output:

```
## Root Cause Analysis

### What happened
[Error name] in [function name] — [X] occurrences, [criticality] criticality, trend is [trend].
[One sentence describing the failure from the stack trace.]

### Why it happened
[The actual root cause based on evidence. Be specific — not "something went wrong" but "the payment intent ID format changed after the Stripe SDK upgrade deployed at 14:30 UTC".]

### Evidence
- Stack trace: [file:line] — [what the code does at that point]
- CloudWatch logs: [key finding from the logs, OR "SKIPPED — reason" if skipped. Never omit this line.]
- Git history: [relevant commit or "no changes in 48h"]
- Deployments: [deployment event or "none detected"]
- Resource metrics: [memory/timeout/duration finding, OR "SKIPPED — CloudWatch unavailable" if skipped. Never omit this line.]
- Downstream: [dependency finding or "all healthy"]

### Suggested Fix
[Specific code change. Show the actual fix if possible — a code block with the change.
If it's a config issue, show the exact config change.]

### How to Prevent Recurrence
[Concrete actions: add input validation, add a test case, add an alarm, increase timeout, etc.]
```

**If upstream resources exist** (from `data.graph.upstream`), mention what triggers this Lambda (e.g. "This Lambda is invoked by API Gateway endpoint `POST /orders/webhook` via `routes_to` edge").

---

## Step 7: Record Resolution

**Only after the fix has been applied** (code committed, config changed, or developer confirms the issue is handled). Do NOT ask for feedback before the fix is done.

The sequence is:
1. You presented the root cause analysis (Step 6)
2. You wrote the fix code
3. The developer reviewed and approved the fix
4. NOW ask for feedback

Ask the developer:

> "The fix has been applied. Want to record what resolved this? (type your notes, or press Enter to skip)"

**If the developer provides feedback**, send it to Watchy and mark the error as resolved:

```bash
curl -s -X POST -H "Authorization: Bearer $WATCHY_API_KEY" \
  -H "Content-Type: application/json" \
  "$BASE_URL/api/v1/errors/ERROR_GROUP_ID/feedback" \
  -d '{"note": "DEVELOPER_FEEDBACK", "resolve": true}'
```

Replace `ERROR_GROUP_ID` with the error group ID from Step 1. Replace `DEVELOPER_FEEDBACK` with what the developer typed.

**If the developer skips** (empty input), still mark it resolved:

```bash
curl -s -X POST -H "Authorization: Bearer $WATCHY_API_KEY" \
  -H "Content-Type: application/json" \
  "$BASE_URL/api/v1/errors/ERROR_GROUP_ID/feedback" \
  -d '{"note": "", "resolve": true}'
```

**If the POST fails**, don't block — just note that feedback wasn't recorded and move on.

---

## Step 8: Suggest Error Capture (if not installed)

**Only if `WATCHY_CAPTURE_NOT_INSTALLED` flag is set from Step 0.**

After presenting the root cause analysis, add:

```
---

This error wasn't captured by Watchy — I had to search CloudWatch directly.
To catch errors like this automatically next time, add Watchy's error capture
to this Lambda. It takes about 2 minutes:

1. Create a Watchy API key with write:errors scope at watchy.dev -> Settings -> API Keys
2. Store it in AWS Secrets Manager:
   aws secretsmanager create-secret --name watchy/api-key --secret-string "YOUR_KEY" --region REGION
3. Say "set up watchy error capture" and I'll generate the capture code for your handlers.

With capture installed, you get: stack traces with business context, criticality levels,
resource graph showing what calls this Lambda and what it depends on, and deployment
correlation — all without digging through CloudWatch.
```

---

## Quick Reference: Watchy API Endpoints

All requests: `curl -s -H "Authorization: Bearer $WATCHY_API_KEY" "$BASE_URL/..."`

| Endpoint | Returns |
|---|---|
| `GET /api/v1/errors?status=unresolved&time_range=24h` | Error groups with `id`, `errorName`, `functionName`, `eventCount`, `criticality` |
| `GET /api/v1/errors/{id}/investigate` | Full context: error + events + resource config + graph + deployments + history |
| `GET /api/v1/errors/{id}?max_events=5` | Error group detail with recent events (lighter than /investigate) |
| `POST /api/v1/errors/{id}/feedback` | Submit resolution feedback: `{"note": "...", "resolve": true}` |
| `GET /api/v1/errors/lambdas?time_range=24h` | Errors aggregated by Lambda function |
| `GET /api/v1/stacks` | All stacks with resource counts |
| `GET /api/v1/stacks/{id}` | Stack topology: resources grouped by type, dependency edges |

Every response has a `summary` field with a human-readable overview.
