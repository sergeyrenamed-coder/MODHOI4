# Bridge protocol policy

## JSON contract and validation

- Runtime validates every AI response against `bridge/schema/decision_response.schema.json`.
- If JSON parsing fails or schema validation fails, the bridge **must emit `reject`** as the effective action.
- Invalid payloads are logged with `request_id`, validation error list, and a redacted raw body hash for auditability.

## Deterministic fallback for timeout/error

When the model call times out or returns transport/runtime errors, the bridge must apply deterministic fallback:

1. Use synthetic response generated from request fields only (no random sampling).
2. Set `action` to `delay` when available in `allowed_actions`; otherwise set `reject`.
3. Set:
   - `confidence`: `0.0`
   - `reason_codes`: `["fallback_timeout_or_error"]`
   - `cooldown_hint_days`: `3`
   - `requires_human_visible_event`: `true`
4. Keep `request_id` identical to input request.

This guarantees replay-safe behavior across identical saves/seeds.

## Rate limit and batching

- Per-country limit: max **1 bridge call / game day**.
- Global limit: max **120 bridge calls / minute** across all countries.
- Excess requests are queued and coalesced into deterministic batches keyed by `(turn_id, country_tag)`.
- Batch processing order is stable: ascending `turn_id`, then lexicographic `country_tag`.
- Duplicate requests with the same `request_id` in a batch are de-duplicated by first-seen rule.
- Queue overflow policy: oldest pending request per country is replaced by newest request for the same country (last-write-wins within country lane).
