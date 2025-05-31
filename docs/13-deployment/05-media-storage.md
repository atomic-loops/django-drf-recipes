# Media Storage

## Problem
How do you handle user-uploaded media files (images, documents) in production for a Django REST Framework API?

## Solution
Configure Django's `MEDIA_ROOT` and `MEDIA_URL` for local storage, and use a cloud storage backend (e.g., AWS S3) for production with `django-storages`.

## Code

### settings.py (local storage)
```python
MEDIA_URL = '/media/'
MEDIA_ROOT = '/path/to/your/media/'
```

### Example Nginx config
```
location /media/ {
    alias /path/to/your/media/;
}
```

### Cloud storage with AWS S3
```bash
pip install django-storages boto3
```

#### settings.py (S3)
```python
INSTALLED_APPS += ['storages']
DEFAULT_FILE_STORAGE = 'storages.backends.s3boto3.S3Boto3Storage'
AWS_ACCESS_KEY_ID = 'your-access-key'
AWS_SECRET_ACCESS_KEY = 'your-secret-key'
AWS_STORAGE_BUCKET_NAME = 'your-bucket-name'
AWS_S3_REGION_NAME = 'your-region'
MEDIA_URL = f'https://{AWS_STORAGE_BUCKET_NAME}.s3.amazonaws.com/'
```

## Notes
- Never serve media files with Django in production; use Nginx or a CDN.
- For large files or high-traffic sites, always use cloud storage or a CDN for scalability and reliability.
- Protect sensitive media files with appropriate permissions and signed URLs if needed. 