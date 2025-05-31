# CI/CD Pipeline

## Problem
How do you automate testing, building, and deployment of your Django REST Framework project?

## Solution
Set up a CI/CD pipeline using GitHub Actions to run tests, lint code, and deploy on every push.

## Code

### .github/workflows/ci.yml
```yaml
name: Django CI

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:13
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: test_db
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    env:
      DATABASE_URL: postgres://postgres:postgres@localhost:5432/test_db
      SECRET_KEY: test
      DEBUG: 'False'
    steps:
    - uses: actions/checkout@v2
    - name: Set up Python
      uses: actions/setup-python@v2
      with:
        python-version: '3.10'
    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt
    - name: Run migrations
      run: |
        python manage.py migrate --noinput
    - name: Run tests
      run: |
        python manage.py test
```

## Notes
- Adjust the workflow for your database and deployment needs.
- Add steps for linting, building Docker images, or deploying to your server or cloud provider.
- Use secrets in GitHub Actions for sensitive values. 