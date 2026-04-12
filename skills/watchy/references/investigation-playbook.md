# Error Investigation Playbook

Follow this playbook precisely when investigating a production error. Each step tells you what to run, what fields to extract, and what to do next.

## Tools You Will Use

- **Bash** — curl (Watchy API), aws CLI (CloudWatch), git, grep
- **Read** — open source files from stack traces
- **Grep** — find files related to a Lambda function

## Prerequisites Check

```bash
echo "WATCHY_API_KEY: ${WATCHY_API_KEY:+set}"
```
```bash
aws sts get-caller-identity --query Account --output text 2>&1 | head -1
```

- If `WATCHY_API_KEY` not set → ask for it (watchy.dev -> Settings -> API Keys, needs `read:errors` scope)
- If `aws sts` fails or returns `ExpiredToken` → ask the dev:
  > "AWS CLI isn't configured (or credentials expired). CloudWatch logs often hold the runtime values needed to diagnose. Paste the three `export` commands from your AWS SSO access portal here and I'll retry — or say 'skip' to continue with Watchy API + git only."
  > 
  > When the dev pastes credentials, chain all three `export` commands to every subsequent `aws` call (`export ... && export ... && export ... && aws logs ...`). Re-run `aws sts get-caller-identity` to verify.
- If the error's `awsAccountId` (from `/investigate`) doesn't match the current CLI account → ask the dev to paste creds for the right account or skip CloudWatch.

### Resolve Base URL

```bash
curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/api/v1/stacks 2>/dev/null
```
Returns `401` or `200` → use `http://localhost:3000`. Otherwise → use `https://watchy.dev`.

---

## Step 1: Get Investigation Context

```bash
curl -s -H "Authorization: Bearer $WATCHY_API_KEY" \
  "$BASE_URL/api/v1/errors/ERROR_GROUP_ID/investigate"
```

If no error group ID, search first:
```bash
curl -s -H "Authorization: Bearer $WATCHY_API_KEY" \
  "$BASE_URL/api/v1/errors?time_range=24h&status=unresolved&include_traces=true&limit=20"
```

**Extract these fields from the /investigate response:**

| Field | JSON path | Used in |
|---|---|---|
| Log group | `data.resource.config.logGroup` | Step 2, 3 |
| Request ID | `data.error.recentEvents[0].requestId` | Step 2, 3 |
| Region | `data.error.group.awsRegion` | Step 2, 3, 5 |
| Memory limit | `data.resource.config.memorySize` | Step 3 |
| Timeout | `data.resource.config.timeout` | Step 3 |
| Stack trace | `data.error.recentEvents[0].stackTrace` | Source file |
| Cold start | `data.error.recentEvents[0].coldStart` | Step 3 |
| Downstream deps | `data.graph.downstream[]` (name, type) | Step 5 |
| Deployments | `data.recentDeployments[]` (eventName, eventTime) | Step 4 |
| Trend | `data.history.trend` | Step 6 |

Read the stack trace → find the source file → use **Read** to open it (strip `/var/task/` prefix, try `.ts` if `.js`).

**Error handling:**
- `401` → bad API key, ask developer
- `404` → wrong error group ID, search again
- Network error → proceed with data from the prompt, skip API steps

---

## Step 2: Pull CloudWatch Logs

**Requires:** `logGroup` and `requestId` from Step 1. Skip if missing.

```bash
aws logs filter-log-events \
  --log-group-name "LOG_GROUP" \
  --filter-pattern "REQUEST_ID" \
  --region REGION
```

- `AccessDeniedException` → note it, skip to Step 4
- `ResourceNotFoundException` → log group doesn't exist, skip to Step 4

Look for: error messages, DB failures, HTTP timeouts, permission errors.

---

## Step 3: Check Execution Metadata

```bash
aws logs filter-log-events \
  --log-group-name "LOG_GROUP" \
  --filter-pattern "REPORT RequestId: REQUEST_ID" \
  --region REGION
```

| Condition | Diagnosis |
|---|---|
| Max Memory Used > 90% of `memorySize` | OOM — increase memory |
| Duration > 80% of `timeout` × 1000 | Timeout — find slow operation |
| Init Duration > 5000ms | Slow cold start — lazy-load SDKs |

---

## Step 4: Check Git History

Find files related to the function:
```bash
grep -rl "FUNCTION_NAME" --include="*.ts" --include="*.js" . 2>/dev/null | head -10
```

Check recent changes:
```bash
git log --since="48 hours ago" --oneline -- MATCHING_FILES
```

Correlate: if a deployment from `recentDeployments` happened within 1 hour of the error's first occurrence, it's the likely cause.

---

## Step 5: Check Downstream Dependencies

For each entry in `data.graph.downstream`, check by `type`:

- **dynamodb_table** → check `ThrottledRequests` metric
- **sqs_queue** → check `ApproximateNumberOfMessages`
- **s3_bucket** → check CloudWatch logs for `AccessDenied`
- Other → note it, check CloudWatch logs

---

## Step 6: Root Cause Analysis

```
## Root Cause Analysis

### What happened
[Error] in [function] — [count] occurrences, [criticality], trend [trend].

### Why it happened
[Specific root cause with evidence.]

### Evidence
- Stack trace: [file:line]
- CloudWatch: [key finding]
- Git: [recent changes or "none in 48h"]
- Deployments: [deployment or "none"]
- Dependencies: [finding or "healthy"]

### Suggested Fix
[Code change or config change.]

### Prevention
[Tests, monitoring, config improvements.]
```

---

## Step 7: Record Resolution

**Only after the fix has been applied** (code committed or developer confirms it's handled).

Ask: "The fix has been applied. Want to record what resolved this? (type your notes, or press Enter to skip)"

If feedback provided:
```bash
curl -s -X POST -H "Authorization: Bearer $WATCHY_API_KEY" \
  -H "Content-Type: application/json" \
  "$BASE_URL/api/v1/errors/ERROR_GROUP_ID/feedback" \
  -d '{"note": "DEVELOPER_FEEDBACK", "resolve": true}'
```

If skipped, still resolve:
```bash
curl -s -X POST -H "Authorization: Bearer $WATCHY_API_KEY" \
  -H "Content-Type: application/json" \
  "$BASE_URL/api/v1/errors/ERROR_GROUP_ID/feedback" \
  -d '{"note": "", "resolve": true}'
```

---

## Step 8: Suggest Error Capture

Only if you had to use CloudWatch directly (error wasn't in Watchy):

> To catch errors like this automatically, add Watchy error capture:
> 1. Create API key at watchy.dev -> Settings -> API Keys (write:errors scope)
> 2. `aws secretsmanager create-secret --name watchy/api-key --secret-string "KEY" --region REGION`
> 3. Say "set up watchy error capture" to generate the handler code.
