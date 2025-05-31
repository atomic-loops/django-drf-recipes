# Static Files

## Problem
How do you serve static files (CSS, JS, images) in production for a Django REST Framework API?

## Solution
Use Django's `collectstatic` command to gather static files, and serve them with Nginx or a CDN in production.

## Code

### settings.py (snippet)
```python
STATIC_URL = '/static/'
STATIC_ROOT = '/path/to/your/static/'
```

### Collect static files (before deployment)
```bash
python manage.py collectstatic --noinput
```

### Example Nginx config
```
location /static/ {
    alias /path/to/your/static/;
}
```

## Notes
- Never use Django's static file server (`runserver`) in production.
- Use a CDN for better performance and scalability.
- Make sure `STATIC_ROOT` is writable by the user running `collectstatic`. 