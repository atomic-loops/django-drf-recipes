# Gunicorn + Nginx Setup

## Problem
How do you serve your Django REST Framework API in production with a robust, performant web server stack?

## Solution
Use Gunicorn as the WSGI application server and Nginx as a reverse proxy and static file server.

## Code

### Install Gunicorn
```bash
pip install gunicorn
```

### Run Gunicorn (development/test)
```bash
gunicorn project.wsgi:application --bind 0.0.0.0:8000
```

### Example Nginx config
```
server {
    listen 80;
    server_name yourdomain.com;

    location /static/ {
        alias /path/to/your/static/;
    }
    location /media/ {
        alias /path/to/your/media/;
    }
    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Systemd service for Gunicorn (optional)
```
[Unit]
Description=gunicorn daemon
After=network.target

[Service]
User=youruser
Group=www-data
WorkingDirectory=/path/to/your/project
ExecStart=/path/to/venv/bin/gunicorn project.wsgi:application --bind 127.0.0.1:8000

[Install]
WantedBy=multi-user.target
```

## Notes
- Always run Gunicorn behind a reverse proxy like Nginx.
- Use `collectstatic` to gather static files before deployment.
- For HTTPS, use Certbot or another tool to configure SSL with Nginx. 