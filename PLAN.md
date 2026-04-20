# Plan: AWS Cloud Security Assessment Skill (`/think-cnap-aws`)

## Context

A Claude Code skill that uses read-only AWS credentials to assess a customer's AWS environment against the think-cnap security framework. The skill fetches the user's existing maturity data and measures from thinkcnap.org, runs AWS CLI checks, and submits updated scores.

**Initial scope: Detection domain (SEC04) only.**

---

## Maturity Scale

| Level | Meaning |
|---|---|
| `-1` | Not applicable — control is irrelevant to this environment |
| `0` | No adoption — no security practice exists |
| `1` | Low adoption — ad-hoc activities, no established security process |
| `2` | Medium adoption — security process exists but not applied to all assets |
| `3` | High adoption — security process exists and applied at scale |

**Field definitions:**
- `initial_maturity` — maturity level before any improvement projects started; used to track year-over-year progress; set once, never overwritten by the skill
- `present_maturity` — current actual maturity level assessed by the skill via AWS CLI
- `desired_maturity` — target goal; should be ambitious but realistic given team size, budget, and tooling choice (open-source vs AWS native vs commercial like Wiz/Orca)

---

## Endpoints

| Purpose | Endpoint |
|---|---|
| Fetch existing measures + user maturity | `GET https://thinkcnap.org/api/integrations/get-user-aws-maturity` |
| Submit updated assessment | `POST https://thinkcnap.org/api/ai-agent-assessment` |

Both endpoints require: `Authorization: Bearer <api_token>`

---

## Files to Create

| File | Action |
|---|---|
| `ai-security-skills/.claude/commands/think-cnap-aws.md` | Create |
| `ai-security-skills/aws-assess-readonly-policy.json` | Create |

---

## Skill Flow

### Step 1 — Collect inputs

Ask the user for:
1. AWS Access Key ID
2. AWS Secret Access Key
3. AWS Region (default: `us-east-1`)
4. think-cnap API token (from thinkcnap.org → Integrations → API Token)

> Advise user to use a dedicated read-only IAM role, not root or admin credentials.

---

### Step 2 — Fetch existing measures and maturity from thinkcnap.org

```bash
curl -s "https://thinkcnap.org/api/integrations/get-user-aws-maturity" \
  -H "Authorization: Bearer <API_TOKEN>"
```

Filter: keep only measures from the Detection domain (`name` contains "Detection").

Build a flat list:
```
[
  { measure_id, measure, comment, impact, effort,
    initial_maturity, present_maturity, desired_maturity }
]
```

Detection domain measures (SEC04):

| measure_id | Description |
|---|---|
| SEC04-BP01-AWS-001 | Configure AWS CloudTrail |
| SEC04-BP01-AWS-002 | Configure OS logging |
| SEC04-BP02-AWS-001 | Implement SIEM |
| SEC04-BP03-AWS-001 | Monitor with GuardDuty |
| SEC04-BP04-AWS-001 | Implement CSPM (Security Hub / Config) |
| SEC04-BP04-AWS-002 | Implement AWS Security Policies (SCPs) |
| SEC04-BP04-AWS-003 | Implement IaC Security Scanner |

---

### Step 3 — Set AWS credentials and verify access

```bash
export AWS_ACCESS_KEY_ID="..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_DEFAULT_REGION="..."
aws sts get-caller-identity
```

If this fails, stop and report the credential error.

---

### Step 4 — Run AWS checks per measure

**SEC04-BP01-AWS-001 — CloudTrail**
```bash
aws cloudtrail describe-trails --include-shadow-trails
aws cloudtrail get-trail-status --name <trailArn>
aws cloudtrail get-event-selectors --trail-name <trailArn>
```
| Score | Condition |
|---|---|
| -1 | Not applicable |
| 0 | No trail exists |
| 1 | Single-region trail only (ad-hoc, limited coverage) |
| 2 | Multi-region trail but no org-wide coverage or missing log validation |
| 3 | Org-wide trail with log file validation and data events enabled |

**SEC04-BP01-AWS-002 — OS logging**
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
| 3 | OS logging on all hosts, centralized and actively monitored |

**SEC04-BP02-AWS-001 — SIEM**
```bash
aws logs describe-log-groups --max-items 5
aws logs describe-subscription-filters --log-group-name <group> 2>/dev/null
```
| Score | Condition |
|---|---|
| -1 | Not applicable |
| 0 | No log aggregation or SIEM |
| 1 | Logs stored in S3 only, no correlation |
| 2 | CloudWatch Logs or basic aggregation, limited correlation |
| 3 | Full SIEM (OpenSearch / Security Lake / third-party) with active alerting |

**SEC04-BP03-AWS-001 — GuardDuty**
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

**SEC04-BP04-AWS-001 — CSPM (Security Hub / Config)**
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
| 0 | Neither Security Hub nor Config enabled |
| 1 | AWS Config enabled only, no Security Hub |
| 2 | Security Hub enabled with at least one standard (CIS/NIST) |
| 3 | Security Hub org-wide + conformance packs + automated remediation |

**SEC04-BP04-AWS-002 — AWS Security Policies (SCPs)**
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

**SEC04-BP04-AWS-003 — IaC Security Scanner**
```bash
# Check CI/CD evidence — ask user to confirm if IaC scanning exists in pipelines
```
| Score | Condition |
|---|---|
| -1 | No IaC used |
| 0 | No IaC security scanning |
| 1 | Scanner run ad-hoc, not in CI/CD |
| 2 | Scanner in CI/CD but in warn-only mode |
| 3 | Scanner in CI/CD in blocking mode (Trivy / Checkov / KICS) |

---

### Step 5 — Ask about desired maturity context

Before calculating `desired_maturity`, ask:

```
To set a realistic desired maturity target, I need to understand your context:

1. How many cloud security engineers work on AWS security? (e.g. 1, 2-5, 5+)
2. What is your tooling preference?
   [a] Open-source only (Prowler, Trivy, Sigma rules)
   [b] AWS native (GuardDuty, Security Hub, Config)
   [c] Commercial (Wiz, Orca Security, Lacework, etc.)
3. What is the target timeframe? (e.g. 6 months, 12 months)
```

Use the answers to set `desired_maturity`:
- With 1 engineer + open-source + 12 months → typically achievable: 2 for complex measures, 3 for lightweight ones
- With 2-5 engineers + AWS native + 12 months → typically achievable: 3 for most measures
- With commercial tooling → faster path to 3 on detection/CSPM measures

---

### Step 6 — Calculate final maturity values

For each assessed measure:
```
present_maturity = score from Step 4

# Preserve initial_maturity if already set (>= 0); only set on first assessment
if existing.initial_maturity == -1:
  initial_maturity = present_maturity
else:
  initial_maturity = existing.initial_maturity   # never overwrite

desired_maturity = value from Step 5 context (not a formula — a judgement)
```

---

### Step 7 — Ask user to review before submitting

```
## Detection Domain — Assessment Results

| measure_id | Description | Initial | Present | Desired | Finding |
|---|---|---|---|---|---|
| SEC04-BP01-AWS-001 | CloudTrail | 1 | 2 | 3 | Org-wide trail missing |
| SEC04-BP03-AWS-001 | GuardDuty  | 0 | 0 | 3 | Not enabled |

Would you like to:
  [1] Submit these results to thinkcnap.org
  [2] Edit a value before submitting (type: edit SEC04-BP01-AWS-001 present=3)
  [3] Cancel
```

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
        { "measure_id": "SEC04-BP03-AWS-001", "title": "Enable GuardDuty org-wide", "impact": "high", "effort": "low" }
      ]
    }
  }'
```

---

### Step 9 — Report to user

Output:
- Table of measures assessed with all three maturity levels
- Key risks (present_maturity = 0)
- Top recommendations sorted by high impact + low effort first
- Progress delta: measures where present > initial (improvements since baseline)
- Link: `https://thinkcnap.org` to view the full assessment

---

## IAM Policy

**File:** `ai-security-skills/aws-assess-readonly-policy.json`

Minimal read-only permissions for Detection domain checks:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DetectionReadOnly",
      "Effect": "Allow",
      "Action": [
        "sts:GetCallerIdentity",
        "cloudtrail:DescribeTrails",
        "cloudtrail:GetTrailStatus",
        "cloudtrail:GetEventSelectors",
        "ec2:DescribeVpcs",
        "ec2:DescribeFlowLogs",
        "route53resolver:ListResolverQueryLogConfigs",
        "logs:DescribeLogGroups",
        "logs:DescribeSubscriptionFilters",
        "ssm:DescribeInstanceInformation",
        "guardduty:ListDetectors",
        "guardduty:GetDetector",
        "guardduty:ListOrganizationAdminAccounts",
        "guardduty:GetOrganizationConfiguration",
        "securityhub:DescribeHub",
        "securityhub:GetEnabledStandards",
        "config:DescribeConfigurationRecorders",
        "config:DescribeConfigurationRecorderStatus",
        "config:DescribeConformancePacks",
        "organizations:DescribeOrganization",
        "organizations:ListPolicies"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## Future Scope (not in v1)

- Expand to all 7 domains
- Multi-account assessment
- Delta report: show improvements since initial_maturity baseline
