# Database Migrations

## Problem
How do you safely apply database schema changes (migrations) when deploying Django REST Framework APIs to production?

## Solution
Use Django's built-in migration system. Run `makemigrations` during development and `migrate` during deployment. Automate migrations in your deployment pipeline.

## Code

### Create migrations (development)
```bash
python manage.py makemigrations
```

### Apply migrations (production/deployment)
```bash
python manage.py migrate --noinput
```

### Example: Deploy script snippet
```bash
# Pull new code
git pull
# Install dependencies
pip install -r requirements.txt
# Apply migrations
python manage.py migrate --noinput
# Collect static files
python manage.py collectstatic --noinput
# Restart app server
sudo systemctl restart gunicorn
```

## Notes
- Always back up your database before applying migrations in production.
- Use `--noinput` for non-interactive/automated deployments.
- Review generated migrations before applying, especially for destructive changes. 