---
type: always_apply
---

# Privacy & Security Rules

## Never Commit to Git

- `.env` files, API keys, tokens, or secrets of any kind
- Database credentials or connection strings
- Private keys or certificates
- Personal user data or PII

## Security Standards

- Validate all user inputs; sanitize all outputs before rendering
- Use environment variables for all configuration (never hardcode credentials)
- Implement proper authentication before authorization checks
- Follow OWASP Top 10 guidelines for web security
- Use HTTPS for all external API calls
- Apply least-privilege principle for all permissions

## File Access

- When accessing potentially sensitive files (`.env`, config files), inform the user of intent
- Do not expose secrets in logs, error messages, comments, or debug output
- Review staged files before committing to ensure no credentials are included
- If blocked from accessing a sensitive file, ask user for explicit approval before proceeding

