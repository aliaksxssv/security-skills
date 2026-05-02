# think-cnap API Reference

Base URL: `https://thinkcnap.org`  
Auth: `Authorization: Bearer <token>` — get from thinkcnap.org → Integrations → API Token.  
Errors: `401 { "error": "..." }` for missing/invalid/expired/revoked token.

## GET /api/integrations/get-user-aws-maturity

Returns all AWS-tagged measures across all domains with current scores.

Response schema:
```json
{
  "generated_at": "<ISO8601>",
  "domains": [{
    "id": 4,
    "name": "Detection",
    "controls": [{
      "id": 7,
      "code": "SEC04-BP01",
      "description": "Configure service and application logging",
      "measures": [{
        "measure_id": "SEC04-BP01-AWS-001",
        "measure": "Configure CloudTrail",
        "comment": "",
        "impact": "high",
        "effort": "low",
        "initial_maturity": -1,
        "present_maturity": -1,
        "desired_maturity": -1
      }]
    }]
  }]
}
```

## POST /api/integrations/update-measure

Submit a score for a single measure. Call once per measure. Omit measures where `present_maturity` is `-1` (not applicable) or skipped due to insufficient evidence.

```bash
curl -s -X POST "https://thinkcnap.org/api/integrations/update-measure" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <TOKEN>" \
  -d '{
    "measure_id": "SEC04-BP01-AWS-001",
    "impact": "<from GET response>",
    "effort": "<from GET response>",
    "initial_maturity": 1,
    "present_maturity": 2,
    "desired_maturity": 3
  }'
```

Check the response after each call and report any errors before continuing.
