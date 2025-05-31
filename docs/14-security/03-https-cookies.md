# HTTPS-only Cookies

## Problem
How do you ensure cookies (e.g., sessionid, csrftoken) are only sent over secure HTTPS connections in Django REST Framework APIs?

## Solution
Set the `SESSION_COOKIE_SECURE` and `CSRF_COOKIE_SECURE` settings to `True` in production. Optionally, set `SECURE_HSTS_SECONDS` and `SECURE_SSL_REDIRECT` for extra security.

## Code

### settings.py (production)
```python
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SECURE_HSTS_SECONDS = 31536000  # 1 year
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_SSL_REDIRECT = True
```

## Notes
- These settings ensure cookies are only sent over HTTPS, protecting them from interception.
- Set these only in production; in development, you may use HTTP for convenience.
- Use `SECURE_HSTS_SECONDS` to enable HTTP Strict Transport Security (HSTS) for your domain. 