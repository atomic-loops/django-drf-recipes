# Environment Variable Management

## Problem
How do you securely manage sensitive settings (e.g., secrets, database URLs) in Django for different environments (development, staging, production)?

## Solution
Use environment variables to store secrets and configuration, and load them in your Django settings. Use `python-dotenv` for local development.

## Code

### Install python-dotenv (optional, for local dev)
```bash
pip install python-dotenv
```

### .env file (do not commit to version control)
```
DEBUG=False
SECRET_KEY=your-secret-key
DATABASE_URL=postgres://user:pass@localhost:5432/dbname
```

### settings.py (snippet)
```python
import os
from pathlib import Path
from dotenv import load_dotenv

BASE_DIR = Path(__file__).resolve().parent.parent
load_dotenv(BASE_DIR / '.env')

SECRET_KEY = os.environ.get('SECRET_KEY')
DEBUG = os.environ.get('DEBUG', 'False') == 'True'
DATABASE_URL = os.environ.get('DATABASE_URL')
```

## Notes
- Never commit your `.env` file or secrets to version control.
- Use environment variables for all secrets and environment-specific settings.
- On production, set environment variables via your process manager, container, or cloud provider. 