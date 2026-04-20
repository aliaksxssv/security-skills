Assess an AWS environment against the think-cnap Detection domain security framework (SEC04), then submit the maturity scores to thinkcnap.org.

## Maturity Scale

| Level | Meaning |
|---|---|
| `-1` | Not applicable — control is irrelevant to this environment |
| `0` | No adoption — no security practice exists |
| `1` | Low adoption — ad-hoc activities, no established process |
| `2` | Medium adoption — process exists but not applied to all assets |
| `3` | High adoption — process exists and applied at scale |

**Fields:**
- `initial_maturity` — baseline before any improvement projects; tracks year-over-year progress; set once, never overwritten
- `present_maturity` — current actual state assessed via AWS CLI
- `desired_maturity` — realistic goal given team size, budget, and tooling

## Instructions

### Step 1 — Collect inputs

Ask the user for:
1. AWS Access Key ID
2. AWS Secret Access Key
3. AWS Region (default: `us-east-1`)
4. think-cnap API token (from thinkcnap.org → Integrations → API Token)

> Recommend using a dedicated read-only IAM role. See `aws-assess-readonly-policy.json` for the required permissions.

---

### Step 2 — Fetch existing measures and maturity

```bash
curl -s "https://thinkcnap.org/api/integrations/get-user-aws-maturity" \
  -H "Authorization: Bearer <API_TOKEN>"
```

Filter the response: keep only the domain where `name` contains `"Detection"`.

Build a flat list of measures with their existing scores:
```
[{ measure_id, measure, comment, impact, effort, initial_maturity, present_maturity, desired_maturity }]
```

If the API returns an error, stop and report it to the user.

---

### Step 3 — Set AWS credentials and verify access

```bash
export AWS_ACCESS_KEY_ID="<key>"
export AWS_SECRET_ACCESS_KEY="<secret>"
export AWS_DEFAULT_REGION="<region>"
aws sts get-caller-identity
```

Save the `Account` value from the response — this is the `aws_account_id` for submission.

If this fails, stop and report the credential error.

---

### Step 4 — Run AWS checks per measure

Run all checks below, then assign a maturity score using the rubric for each measure.

---

**SEC04-BP01-AWS-001 — Configure AWS CloudTrail**
```bash
aws cloudtrail describe-trails --include-shadow-trails
# For each trail ARN:
aws cloudtrail get-trail-status --name <trailArn>
aws cloudtrail get-event-selectors --trail-name <trailArn>
```
| Score | Condition |
|---|---|
| -1 | Not applicable |
| 0 | No trail exists |
| 1 | Single-region trail only |
| 2 | Multi-region trail, but not org-wide or missing log file validation |
| 3 | Org-wide trail with log file validation and data events enabled |

---

**SEC04-BP01-AWS-002 — Configure OS logging**
```bash
aws ssm describe-instance-information 2>/dev/null
aws logs describe-log-groups --log-group-name-prefix /aws/ec2 2>/dev/null
```
| Score | Condition |
|---|---|
| -1 | No EC2 or managed compute in this account |
| 0 | No OS-level logging configured |
| 1 | OS logging on some hosts only (ad-hoc) |
| 2 | OS logging on most hosts, not centralized |
| 3 | OS logging on all hosts, centralized and monitored |

---

**SEC04-BP02-AWS-001 — Implement SIEM**
```bash
aws logs describe-log-groups --max-items 5
# For a representative log group:
aws logs describe-subscription-filters --log-group-name <group> 2>/dev/null
```
| Score | Condition |
|---|---|
| -1 | Not applicable |
| 0 | No log aggregation or SIEM |
| 1 | Logs stored in S3 only, no correlation |
| 2 | CloudWatch Logs or basic aggregation, limited correlation |
| 3 | Full SIEM (OpenSearch / Security Lake / third-party) with active alerting |

---

**SEC04-BP03-AWS-001 — Monitor cloud environment (GuardDuty)**
```bash
aws guardduty list-detectors
aws guardduty get-detector --detector-id <id>
aws guardduty list-organization-admin-accounts 2>/dev/null
aws guardduty get-organization-configuration --detector-id <id> 2>/dev/null
```
| Score | Condition |
|---|---|
| -1 | Not applicable |
| 0 | GuardDuty not enabled |
| 1 | Enabled in one region only, findings not actioned |
| 2 | Enabled in multiple regions, basic alerting |
| 3 | Org-wide with S3/EKS/RDS protection and automated response |

---

**SEC04-BP04-AWS-001 — Implement CSPM (Security Hub / Config)**
```bash
aws securityhub describe-hub 2>/dev/null
aws securityhub get-enabled-standards 2>/dev/null
aws config describe-configuration-recorders
aws config describe-configuration-recorder-status
aws config describe-conformance-packs 2>/dev/null
```
| Score | Condition |
|---|---|
| -1 | Not applicable |
| 0 | Neither Security Hub nor AWS Config enabled |
| 1 | AWS Config enabled only, no Security Hub |
| 2 | Security Hub enabled with at least one standard (CIS/NIST) |
| 3 | Security Hub org-wide + conformance packs + automated remediation |

---

**SEC04-BP04-AWS-002 — Implement AWS Security Policies (SCPs)**
```bash
aws organizations describe-organization 2>/dev/null
aws organizations list-policies --filter SERVICE_CONTROL_POLICY 2>/dev/null
```
| Score | Condition |
|---|---|
| -1 | Not using AWS Organizations |
| 0 | No SCPs applied |
| 1 | A few SCPs applied manually, no systematic approach |
| 2 | SCPs applied to most OUs covering key controls |
| 3 | Comprehensive SCP set applied org-wide, version-controlled |

---

**SEC04-BP04-AWS-003 — Implement IaC Security Scanner**

Ask the user: "Do you use IaC (Terraform, CloudFormation, CDK)? If yes, is a security scanner (Trivy, Checkov, KICS) integrated in your CI/CD pipeline?"

| Score | Condition |
|---|---|
| -1 | No IaC used |
| 0 | No IaC security scanning |
| 1 | Scanner run ad-hoc, not in CI/CD |
| 2 | Scanner in CI/CD in warn-only mode |
| 3 | Scanner in CI/CD in blocking mode |

---

### Step 5 — Ask about desired maturity context

Ask the user:

```
To set realistic desired maturity targets, I need to understand your context:

1. How many cloud security engineers work on AWS security? (1 / 2–5 / 5+)
2. Tooling preference?
   [a] Open-source (Prowler, Trivy, Sigma rules)
   [b] AWS native (GuardDuty, Security Hub, Config)
   [c] Commercial (Wiz, Orca Security, Lacework, etc.)
3. Target timeframe? (6 months / 12 months / 18+ months)
```

Set `desired_maturity` per measure as a judgement based on the answers:
- 1 engineer + open-source + 6 months → target 2 for complex measures, 3 for lightweight ones
- 2–5 engineers + AWS native + 12 months → target 3 for most measures
- Commercial tooling → faster path to 3 on detection and CSPM measures

---

### Step 6 — Calculate final maturity values

For each measure:
```
present_maturity  = score from Step 4

# Preserve initial_maturity if already set — it tracks the baseline
if existing.initial_maturity is -1 or 0 and was never intentionally set:
  initial_maturity = present_maturity
else:
  initial_maturity = existing.initial_maturity   # never overwrite

desired_maturity  = value from Step 5
```

---

### Step 7 — Ask user to review before submitting

Show a summary table:

```
## Detection Domain — Assessment Results

| measure_id          | Description          | Initial | Present | Desired | Key Finding                  |
|---------------------|----------------------|---------|---------|---------|------------------------------|
| SEC04-BP01-AWS-001  | CloudTrail           | 1       | 2       | 3       | Org-wide trail missing       |
| SEC04-BP03-AWS-001  | GuardDuty            | 0       | 0       | 3       | Not enabled                  |
| SEC04-BP04-AWS-003  | IaC Scanner          | -1      | -1      | -1      | No IaC used                  |

Would you like to:
  [1] Submit these results to thinkcnap.org
  [2] Edit a value before submitting (type: edit SEC04-BP01-AWS-001 present=3)
  [3] Cancel
```

Apply any edits, then proceed.

---

### Step 8 — Submit to thinkcnap.org

```bash
curl -s -X POST "https://thinkcnap.org/api/ai-agent-assessment" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <API_TOKEN>" \
  -d '{
    "aws_account_id": "<from sts get-caller-identity>",
    "measures": {
      "SEC04-BP01-AWS-001": {
        "impact": "high",
        "effort": "low",
        "initial_maturity": 1,
        "present_maturity": 2,
        "desired_maturity": 3
      }
    },
    "summary": {
      "domain": "Detection",
      "measures_assessed": 7,
      "key_risks": ["GuardDuty not enabled", "No SIEM"],
      "top_recommendations": [
        {
          "measure_id": "SEC04-BP03-AWS-001",
          "title": "Enable GuardDuty org-wide",
          "impact": "high",
          "effort": "low"
        }
      ]
    }
  }'
```

Omit measures where `present_maturity` is `-1` (not applicable) from the submission.

---

### Step 9 — Report to user

Output:
- Table with all three maturity levels per measure
- Key risks: measures where `present_maturity` is `0`
- Top recommendations sorted by `impact: high` + `effort: low` first
- Progress: measures where `present_maturity > initial_maturity` (improvements since baseline)
- Link to view the full assessment: `https://thinkcnap.org`
