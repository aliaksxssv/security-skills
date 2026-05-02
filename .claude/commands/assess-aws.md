Assess an AWS environment against the think-cnap security framework and submit maturity scores to thinkcnap.org.

## Maturity Scale

Read `memory/maturity-scale.md` for the full scoring rubric, field definitions, and guidance on setting `desired_maturity`.

## Instructions

### Step 1 — Collect inputs

Read `memory/user-profile.md` first. It may contain a `## Credentials` section with:

- `thinkcnap.org API token`
- `AWS Access Key ID`
- `AWS Secret Access Key`
- `AWS Region`

Use any values found there silently — do not ask for credentials that are already present. Only prompt for credentials that are missing.

If you collect new credentials from the user, save them to the `## Credentials` section in `memory/user-profile.md` (create the section if absent). Never display credentials in terminal output — mask as `***` if they must be referenced.

Once all credentials are known, export them as environment variables before running any commands:

```bash
export THINKCNAP_API_TOKEN="<token>"
export AWS_ACCESS_KEY_ID="<key>"
export AWS_SECRET_ACCESS_KEY="<secret>"
export AWS_DEFAULT_REGION="<region>"
```

Use `$THINKCNAP_API_TOKEN`, `$AWS_ACCESS_KEY_ID`, etc. in all subsequent shell commands — never inline the values.

> Attach the AWS managed policy `arn:aws:iam::aws:policy/SecurityAudit` to the IAM user or role. It grants read-only access across all services this assessment needs.

The file also contains company context, tooling answers with dates, and prior assessment findings. Note what is already known so you do not ask for it again unless it is stale (see re-ask rules in Steps 4b and 5).

---

### Step 2 — Fetch framework from thinkcnap.org

```bash
curl -s "https://thinkcnap.org/api/integrations/get-user-aws-maturity" \
  -H "Authorization: Bearer $THINKCNAP_API_TOKEN"
```

If the API returns an error, stop and report it to the user.

Parse the response and build a list of all available domains with their controls and measures.

---

### Step 3 — Verify AWS access

```bash
aws sts get-caller-identity
```

Save the `Account` value — this is the `aws_account_id` for submission.

If this fails, stop and report the credential error.

---

### Step 3b — Ask user what to assess

Use a drill-down flow. Present one level at a time and wait for the user's answer before going deeper.

**Level 1 — Domain selection**

Show all available domains and ask:
```
Which domain(s) would you like to assess?
  - Enter a number (e.g. "3") or multiple (e.g. "1,3")
  - Enter "all" to assess all domains
```

**Level 2 — Control selection** (ask once per selected domain)

List the controls in that domain and ask:
```
Which controls in <Domain Name> would you like to assess?
  - Enter control codes (e.g. "SEC04-BP01") or multiple (e.g. "SEC04-BP01,SEC04-BP03")
  - Enter "all" to assess all controls in this domain
```

**Level 3 — Measure selection** (only ask if a control has more than one measure)

List the measures in that control and ask:
```
Which measures in <SECXX-BPXX> would you like to assess?
  - Enter measure IDs (e.g. "SEC04-BP01-AWS-001") or multiple
  - Enter "all" to assess all measures in this control
```

If a control has only one measure, skip this step and proceed directly to assessment.

Once scope is confirmed at all levels, build the final flat list of measures to assess. Do not start assessing until the full scope is confirmed.

---

### Step 4 — Assess each measure dynamically

For each measure in the selected list, repeat these three sub-steps:

**4a — Determine what to check**

Read the measure's `measure` name and `comment` from the API response — use both as context to understand what the control requires and what good implementation looks like.

Based on that context, reason about which AWS CLI commands would reveal whether this control is implemented. Consider:
- Which AWS services implement or evidence this control?
- What configuration or status fields indicate adoption level?
- Are there commands that don't require write permissions?

If the measure cannot be assessed via AWS CLI (e.g. it depends on external tooling or process), note that you will ask the user instead.

**4b — Run the checks**

Run the AWS CLI commands you identified. For measures that cannot be assessed via CLI, ask the user a targeted question (e.g. "Do you use a SIEM? If so, which one?").

**Before asking any tooling question**, check `memory/user-profile.md`:
- If the answer exists and was recorded **≤ 1 month ago** — use it silently, do not ask.
- If the answer is **missing or > 1 month old** — ask the user, then save the answer with today's date back to `memory/user-profile.md` under `## Tooling`.

Date format for tooling entries: `(last confirmed: YYYY-MM-DD)`.

**4c — Score the measure**

Only assign a score if you have confident evidence — either clear CLI output or a definitive user answer. Use the measure's `comment` from the API as additional context when interpreting results.

If evidence is ambiguous, incomplete, or the user cannot answer, mark the measure as **skipped** and explain why. Do not guess or infer a score from insufficient data.

Confident evidence means:
- CLI returned clear positive or negative results (e.g. empty list, specific config fields present/absent)
- User gave a direct answer with enough detail to distinguish between maturity levels

Write a one-line `key_finding` explaining the score or the reason for skipping (e.g. "Multi-region trail exists but log file validation is disabled" or "Skipped — user does not know if SIEM is configured").

**4d — Save findings to user-profile.md**

After scoring (or skipping), append the result to `memory/user-profile.md` under `## Assessment findings → AWS account <account_id> — <region>`:

- For scored measures: add a table row and full detail block (see below)
- For skipped measures: add a table row marked `SKIPPED` with the reason — do not submit to thinkcnap.org

1. Add a row to the findings table: `| measure_id | description | initial | present | desired | date |`
2. Add a detail block:
   ```
   #### <measure_id> — <measure name>
   - **Score:** <score> (<label from maturity scale>)
   - **Key finding:** <one-line explanation>
   - **Affected resources:** <ARNs or "None" if not configured>
   - **Why this score:** <which CLI output or user answer drove the decision>
   - **Action items:** prioritized list of concrete remediation steps
   ```

Create the account/region section if it doesn't exist yet.

---

### Step 5 — Determine desired maturity context

`desired_maturity` must be set per measure based on company context. Read `memory/maturity-scale.md` for scoring guidance.

**Check `memory/user-profile.md` for a `## Context` section:**

- If context exists and `last updated` is **≤ 1 month ago** — use it silently, skip to Step 6.
- If context exists but is **> 1 month old** — show saved values and ask the user to confirm or update.
- If context is **missing** — ask the user for all four fields before proceeding:

```
To set realistic desired maturity targets I need your context:

1. Business type?
   [a] Fintech / Banking / Insurance / Healthcare — high compliance, strict targets
   [b] E-commerce / SaaS / Media / Gaming — moderate compliance, balanced targets
   [c] Internal tooling / Early-stage startup / Non-regulated SMB — lighter targets
2. Cloud security engineers working on AWS? (1 / 2–5 / 5+)
3. Security tooling?
   [a] Open-source (Prowler, Trivy, Sigma rules)
   [b] AWS native (GuardDuty, Security Hub, Config)
   [c] Commercial (Wiz, Orca, Lacework, etc.)
4. Improvement timeframe? (6 months / 12 months / 18+ months)
```

After collecting or confirming, save/overwrite the `## Context` section in `memory/user-profile.md`:

```markdown
## Context (last updated: YYYY-MM-DD)

- **Business type:** <value>
- **Security engineers:** <value>
- **Tooling:** <value>
- **Timeframe:** <value>
```

---

### Step 6 — Calculate final maturity values

For each measure:
```
if measure is not applicable:
  initial_maturity = present_maturity = desired_maturity = -1
else:
  present_maturity = score from Step 4
  initial_maturity = existing.initial_maturity if set, else present_maturity  # never overwrite
  desired_maturity = value from Step 5
```

See `memory/maturity-scale.md` for `initial_maturity` preservation rules.

---

### Step 7 — Ask user to review before submitting

Show a summary table:

```
## Assessment Results

| measure_id          | Description          | Initial | Present | Desired | Key Finding                  |
|---------------------|----------------------|---------|---------|---------|------------------------------|
| SEC04-BP01-AWS-001  | CloudTrail           | 1       | 2       | 3       | Log file validation disabled |
| SEC04-BP03-AWS-001  | GuardDuty            | 0       | 0       | 3       | Not enabled                  |

Would you like to:
  [1] Submit these results to thinkcnap.org
  [2] Edit a value before submitting (type: edit SEC04-BP01-AWS-001 present=3)
  [3] Cancel
```

Apply any edits, then proceed.

---

### Step 8 — Submit to thinkcnap.org

Call `POST /api/integrations/update-measure` once per measure. Omit:
- Measures where `present_maturity` is `-1` (not applicable)
- Measures marked **skipped** due to insufficient evidence

```bash
curl -s -X POST "https://thinkcnap.org/api/integrations/update-measure" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $THINKCNAP_API_TOKEN" \
  -d '{
    "measure_id": "<measure_id>",
    "impact": "<from API response>",
    "effort": "<from API response>",
    "initial_maturity": <value>,
    "present_maturity": <value>,
    "desired_maturity": <value>
  }'
```

Check the response for each call and report any errors before continuing to the next measure.

---

### Step 9 — Report to user

Output:
- Table with all three maturity levels per measure
- Key risks: measures where `present_maturity` is `0`
- Top recommendations sorted by `impact: high` + `effort: low` first
- Progress: measures where `present_maturity > initial_maturity` (improvements since baseline)
- Link to view the full assessment: `https://thinkcnap.org`
