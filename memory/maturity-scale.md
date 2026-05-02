# Maturity Scale

| Level | Meaning |
|---|---|
| `-1` | Not applicable — control is irrelevant to this environment |
| `0` | No adoption — no security practice exists |
| `1` | Low adoption — ad-hoc activities, no established process |
| `2` | Medium adoption — process exists but not applied to all assets |
| `3` | High adoption — process exists and applied at scale |

## Fields

- `initial_maturity` — historical baseline; see preservation rules below
- `present_maturity` — current actual state assessed via AWS CLI or interview
- `desired_maturity` — realistic goal given business type, team size, tooling, and timeframe

## Setting desired_maturity

Before scoring, ask the user for the following context:

- **Business type** — industry and any compliance obligations (e.g. fintech, health, SaaS, non-regulated SMB)
- **Security team size** — number of people responsible for security operations
- **Tooling** — security tooling already in use (e.g. AWS native, Wiz, Orca, open-source)
- **Timeframe** — realistic improvement horizon (e.g. 6 or 12 months)

Use the collected context to reason about an appropriate `desired_maturity` per measure. Favour stricter targets when compliance obligations are present or tooling lowers the effort required.

## Preserving initial_maturity

If `initial_maturity` is already set to a non-default value in the API response, **never overwrite it**. It is the historical baseline. Only set it when it hasn't been recorded yet (value is `-1` and was never intentionally assigned, or the measure is being assessed for the first time).
