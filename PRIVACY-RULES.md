# Privacy Rules for `oc-public`

`oc-public` is a public knowledge base.

## Hard Rules (Non-negotiable)

- Never publish real personal identifiers.
- Never publish real Discord user IDs, handles, names, emails, phone numbers.
- Never publish secrets (tokens, keys, passwords, internal URLs, credentials).
- Never publish customer/internal/private project data.

## Allowed Content

- General methods, lessons learned, templates, anonymized examples.
- Redacted snippets only.

## Redaction Standard

Replace sensitive values with placeholders, e.g.:

- `<@USER_ID>`
- `<ROLE_NAME>`
- `<INTERNAL_URL>`
- `<API_KEY>`

## Release Check

Before commit/push:

1. Scan for numeric IDs and mentions.
2. Confirm all examples are anonymized.
3. If uncertain, do not publish.
