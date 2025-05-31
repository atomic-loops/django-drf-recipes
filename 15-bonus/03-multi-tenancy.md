# Multi-tenancy

## Problem
How do you support multiple tenants (customers, organizations) in a single Django REST Framework project?

## Solution
Use a multi-tenancy approach such as schema-based (e.g., `django-tenant-schemas`) or row-level filtering (e.g., organization field on models).

## Code

### Simple row-level multi-tenancy (organization field)
```python
from django.db import models
from django.contrib.auth.models import User

class Organization(models.Model):
    name = models.CharField(max_length=100)

class Project(models.Model):
    name = models.CharField(max_length=100)
    organization = models.ForeignKey(Organization, on_delete=models.CASCADE)

class UserProfile(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE)
    organization = models.ForeignKey(Organization, on_delete=models.CASCADE)
```

### views.py (filter by organization)
```python
from rest_framework import viewsets, permissions
from .models import Project
from .serializers import ProjectSerializer

class ProjectViewSet(viewsets.ModelViewSet):
    serializer_class = ProjectSerializer
    permission_classes = [permissions.IsAuthenticated]

    def get_queryset(self):
        return Project.objects.filter(organization=self.request.user.userprofile.organization)
```

### Schema-based multi-tenancy (django-tenant-schemas)
- Install: `pip install django-tenant-schemas`
- See [django-tenant-schemas docs](https://django-tenant-schemas.readthedocs.io/) for setup and usage.

## Notes
- Always filter data by tenant to prevent data leaks.
- Schema-based multi-tenancy is more complex but offers better isolation.
- Row-level filtering is simple and works for most SaaS use cases. 