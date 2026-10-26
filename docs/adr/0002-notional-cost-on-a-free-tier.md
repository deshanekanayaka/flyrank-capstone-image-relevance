# Costs are recorded as notional cost on a free tier

Every vision call and every embedding call in this project operates on the Gemini free
tier, which charges nothing. Thus an honest cost log records 0.00 for every call, and the
budget guard can never refuse a call. We record real token counts and multiply them by
the published paid-tier rates. The rates are held in a rates table in configuration, and we
store the result in `ai_calls.notional_cost_usd`.

The column name carries the honesty: no money changed hands. The number answers the
question that the project implies, which is the cost of this pipeline at 196,660 products
instead of 50. The number also makes the budget guard testable. Set `MONTHLY_BUDGET_USD`
low, and the next call is refused.

## Consequences

Each `ai_calls` row stores the model name, because a rate belongs to a model and a
precision number is unattributable without it. The rates in configuration carry a comment
that cites the Google pricing page and the date that someone read it.
