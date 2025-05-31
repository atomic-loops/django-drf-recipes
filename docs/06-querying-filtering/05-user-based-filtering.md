# User-Based Filtering

## Problem

You need to filter API results based on the authenticated user's context, permissions, ownership, or relationships, ensuring users only see data they're authorized to access while maintaining clean API design.

## Solution

Implement user-based filtering using DRF's permission system, custom filter backends, and queryset filtering to automatically scope data based on user context without exposing sensitive information.

## Basic User-Based Filtering

```python
# models.py
from django.db import models
from django.contrib.auth.models import User

class Organization(models.Model):
    name = models.CharField(max_length=200)
    members = models.ManyToManyField(User, through='Membership')
    
    def __str__(self):
        return self.name

class Membership(models.Model):
    ROLE_CHOICES = [
        ('admin', 'Admin'),
        ('member', 'Member'),
        ('viewer', 'Viewer'),
    ]
    
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    organization = models.ForeignKey(Organization, on_delete=models.CASCADE)
    role = models.CharField(max_length=20, choices=ROLE_CHOICES)
    joined_at = models.DateTimeField(auto_now_add=True)
    
    class Meta:
        unique_together = ['user', 'organization']

class Project(models.Model):
    STATUS_CHOICES = [
        ('draft', 'Draft'),
        ('active', 'Active'),
        ('completed', 'Completed'),
        ('archived', 'Archived'),
    ]
    
    name = models.CharField(max_length=200)
    description = models.TextField()
    organization = models.ForeignKey(Organization, on_delete=models.CASCADE)
    owner = models.ForeignKey(User, on_delete=models.CASCADE, related_name='owned_projects')
    collaborators = models.ManyToManyField(User, related_name='collaborated_projects', blank=True)
    status = models.CharField(max_length=20, choices=STATUS_CHOICES, default='draft')
    is_public = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)
    
    def __str__(self):
        return self.name

class Task(models.Model):
    PRIORITY_CHOICES = [
        ('low', 'Low'),
        ('medium', 'Medium'),
        ('high', 'High'),
        ('urgent', 'Urgent'),
    ]
    
    title = models.CharField(max_length=200)
    description = models.TextField()
    project = models.ForeignKey(Project, on_delete=models.CASCADE, related_name='tasks')
    assigned_to = models.ForeignKey(User, on_delete=models.SET_NULL, null=True, blank=True)
    created_by = models.ForeignKey(User, on_delete=models.CASCADE, related_name='created_tasks')
    priority = models.CharField(max_length=10, choices=PRIORITY_CHOICES, default='medium')
    due_date = models.DateTimeField(null=True, blank=True)
    completed_at = models.DateTimeField(null=True, blank=True)
    is_completed = models.BooleanField(default=False)
    
    def __str__(self):
        return self.title
```

## Custom Filter Backends

```python
# filters.py
from rest_framework import filters
from django.db.models import Q

class UserOrganizationFilterBackend(filters.BaseFilterBackend):
    """Filter objects based on user's organization membership"""
    
    def filter_queryset(self, request, queryset, view):
        if not request.user.is_authenticated:
            return queryset.none()
        
        # Get user's organizations
        user_orgs = request.user.membership_set.values_list('organization', flat=True)
        
        # Filter based on model type
        model = queryset.model
        
        if hasattr(model, 'organization'):
            return queryset.filter(organization__in=user_orgs)
        elif hasattr(model, 'project') and hasattr(model.project.field.related_model, 'organization'):
            return queryset.filter(project__organization__in=user_orgs)
        
        return queryset

class UserAccessFilterBackend(filters.BaseFilterBackend):
    """Filter objects based on user's access permissions"""
    
    def filter_queryset(self, request, queryset, view):
        if not request.user.is_authenticated:
            # Anonymous users can only see public content
            if hasattr(queryset.model, 'is_public'):
                return queryset.filter(is_public=True)
            return queryset.none()
        
        user = request.user
        model = queryset.model
        
        if model.__name__ == 'Project':
            # Users can see projects they own, collaborate on, or public ones in their org
            user_orgs = user.membership_set.values_list('organization', flat=True)
            
            return queryset.filter(
                Q(owner=user) |  # Own projects
                Q(collaborators=user) |  # Collaborated projects
                Q(is_public=True, organization__in=user_orgs)  # Public in user's orgs
            ).distinct()
        
        elif model.__name__ == 'Task':
            # Users can see tasks in projects they have access to
            accessible_projects = self.get_accessible_projects(user)
            
            return queryset.filter(
                Q(project__in=accessible_projects) |
                Q(assigned_to=user) |  # Tasks assigned to user
                Q(created_by=user)     # Tasks created by user
            ).distinct()
        
        return queryset
    
    def get_accessible_projects(self, user):
        """Get projects the user has access to"""
        from .models import Project
        
        user_orgs = user.membership_set.values_list('organization', flat=True)
        
        return Project.objects.filter(
            Q(owner=user) |
            Q(collaborators=user) |
            Q(is_public=True, organization__in=user_orgs)
        ).distinct()

class UserRoleFilterBackend(filters.BaseFilterBackend):
    """Filter objects based on user's role in organization"""
    
    def filter_queryset(self, request, queryset, view):
        if not request.user.is_authenticated:
            return queryset.none()
        
        user = request.user
        
        # Get user's highest role across organizations
        user_memberships = user.membership_set.select_related('organization')
        
        # Admin users see everything in their organizations
        admin_orgs = user_memberships.filter(role='admin').values_list('organization', flat=True)
        
        if admin_orgs.exists():
            if hasattr(queryset.model, 'organization'):
                return queryset.filter(organization__in=admin_orgs)
            elif hasattr(queryset.model, 'project'):
                return queryset.filter(project__organization__in=admin_orgs)
        
        # Apply normal user filtering
        return UserAccessFilterBackend().filter_queryset(request, queryset, view)
```

## View-Level User Filtering

```python
# views.py
from rest_framework import generics, viewsets
from rest_framework.permissions import IsAuthenticated
from django.db.models import Q
from .models import Project, Task, Organization
from .serializers import ProjectSerializer, TaskSerializer, OrganizationSerializer
from .filters import UserOrganizationFilterBackend, UserAccessFilterBackend

class UserProjectListView(generics.ListAPIView):
    serializer_class = ProjectSerializer
    permission_classes = [IsAuthenticated]
    filter_backends = [UserAccessFilterBackend]
    
    def get_queryset(self):
        """Get projects accessible to the current user"""
        user = self.request.user
        user_orgs = user.membership_set.values_list('organization', flat=True)
        
        return Project.objects.filter(
            Q(owner=user) |
            Q(collaborators=user) |
            Q(is_public=True, organization__in=user_orgs)
        ).select_related('organization', 'owner').prefetch_related('collaborators').distinct()

class UserTaskViewSet(viewsets.ModelViewSet):
    serializer_class = TaskSerializer
    permission_classes = [IsAuthenticated]
    filter_backends = [UserAccessFilterBackend]
    
    def get_queryset(self):
        """Get tasks accessible to the current user"""
        user = self.request.user
        
        # Get accessible projects
        accessible_projects = self.get_accessible_projects(user)
        
        # Filter tasks based on multiple criteria
        queryset = Task.objects.filter(
            Q(project__in=accessible_projects) |
            Q(assigned_to=user) |
            Q(created_by=user)
        ).select_related(
            'project', 'assigned_to', 'created_by', 'project__organization'
        ).distinct()
        
        return queryset
    
    def get_accessible_projects(self, user):
        """Get projects the user has access to"""
        user_orgs = user.membership_set.values_list('organization', flat=True)
        
        return Project.objects.filter(
            Q(owner=user) |
            Q(collaborators=user) |
            Q(is_public=True, organization__in=user_orgs)
        ).distinct()
    
    def perform_create(self, serializer):
        """Set user as creator when creating tasks"""
        serializer.save(created_by=self.request.user)

class MyTasksView(generics.ListAPIView):
    """View for tasks specifically assigned to or created by the user"""
    serializer_class = TaskSerializer
    permission_classes = [IsAuthenticated]
    
    def get_queryset(self):
        user = self.request.user
        filter_type = self.request.query_params.get('filter', 'all')
        
        base_queryset = Task.objects.select_related(
            'project', 'assigned_to', 'created_by'
        )
        
        if filter_type == 'assigned':
            return base_queryset.filter(assigned_to=user, is_completed=False)
        elif filter_type == 'created':
            return base_queryset.filter(created_by=user)
        elif filter_type == 'completed':
            return base_queryset.filter(
                Q(assigned_to=user) | Q(created_by=user),
                is_completed=True
            )
        else:
            # All tasks user has access to
            return base_queryset.filter(
                Q(assigned_to=user) | Q(created_by=user)
            )
```

## Advanced User Context Filtering

### Multi-Tenant Filtering

```python
class TenantFilterMixin:
    """Mixin to filter results based on user's tenant context"""
    
    def get_tenant_queryset(self, queryset):
        """Filter queryset based on user's tenant"""
        user = self.request.user
        
        if not user.is_authenticated:
            return queryset.none()
        
        # Get user's current tenant from session or profile
        tenant = getattr(user, 'current_tenant', None) or \
                getattr(user, 'default_tenant', None)
        
        if not tenant:
            return queryset.none()
        
        # Apply tenant filtering based on model
        model = queryset.model
        
        if hasattr(model, 'tenant'):
            return queryset.filter(tenant=tenant)
        elif hasattr(model, 'organization'):
            return queryset.filter(organization=tenant)
        
        return queryset
    
    def get_queryset(self):
        queryset = super().get_queryset()
        return self.get_tenant_queryset(queryset)

class TenantAwareProjectView(TenantFilterMixin, generics.ListAPIView):
    serializer_class = ProjectSerializer
    queryset = Project.objects.all()
```

### Role-Based Filtering

```python
class RoleBasedFilterMixin:
    """Mixin to filter results based on user's role"""
    
    def get_role_based_queryset(self, queryset):
        """Apply role-based filtering"""
        user = self.request.user
        
        if not user.is_authenticated:
            return queryset.none()
        
        # Get user's roles
        user_roles = self.get_user_roles(user)
        
        if 'admin' in user_roles:
            # Admins see everything in their organizations
            return self.get_admin_queryset(queryset, user)
        elif 'manager' in user_roles:
            # Managers see their team's data
            return self.get_manager_queryset(queryset, user)
        else:
            # Regular users see only their own data
            return self.get_user_queryset(queryset, user)
    
    def get_user_roles(self, user):
        """Get user's roles across organizations"""
        return user.membership_set.values_list('role', flat=True)
    
    def get_admin_queryset(self, queryset, user):
        """Admin-level access"""
        admin_orgs = user.membership_set.filter(
            role='admin'
        ).values_list('organization', flat=True)
        
        if hasattr(queryset.model, 'organization'):
            return queryset.filter(organization__in=admin_orgs)
        return queryset
    
    def get_manager_queryset(self, queryset, user):
        """Manager-level access"""
        # Managers see projects they manage
        managed_projects = Project.objects.filter(
            organization__in=user.membership_set.filter(
                role__in=['admin', 'manager']
            ).values_list('organization', flat=True)
        )
        
        if queryset.model == Task:
            return queryset.filter(project__in=managed_projects)
        elif queryset.model == Project:
            return managed_projects
        
        return queryset
    
    def get_user_queryset(self, queryset, user):
        """Regular user access"""
        if queryset.model == Task:
            return queryset.filter(
                Q(assigned_to=user) | Q(created_by=user)
            )
        elif queryset.model == Project:
            return queryset.filter(
                Q(owner=user) | Q(collaborators=user)
            )
        
        return queryset
```

### Conditional Filtering

```python
class ConditionalUserFilterBackend(filters.BaseFilterBackend):
    """Apply different filters based on user attributes or context"""
    
    def filter_queryset(self, request, queryset, view):
        user = request.user
        
        if not user.is_authenticated:
            return self.get_anonymous_queryset(queryset)
        
        # Apply different filtering based on user type
        if user.is_superuser:
            return queryset  # Superusers see everything
        
        if hasattr(user, 'profile'):
            if user.profile.is_premium:
                return self.get_premium_queryset(queryset, user)
            elif user.profile.is_trial:
                return self.get_trial_queryset(queryset, user)
        
        return self.get_standard_queryset(queryset, user)
    
    def get_anonymous_queryset(self, queryset):
        """Filter for anonymous users"""
        if hasattr(queryset.model, 'is_public'):
            return queryset.filter(is_public=True)
        return queryset.none()
    
    def get_premium_queryset(self, queryset, user):
        """Premium users get expanded access"""
        user_orgs = user.membership_set.values_list('organization', flat=True)
        
        return queryset.filter(
            Q(owner=user) |
            Q(collaborators=user) |
            Q(organization__in=user_orgs)  # See all org content
        ).distinct()
    
    def get_trial_queryset(self, queryset, user):
        """Trial users get limited access"""
        return queryset.filter(
            Q(owner=user) |
            Q(assigned_to=user)
        ).distinct()[:10]  # Limit to 10 items
    
    def get_standard_queryset(self, queryset, user):
        """Standard user access"""
        return queryset.filter(
            Q(owner=user) |
            Q(collaborators=user) |
            Q(assigned_to=user)
        ).distinct()
```

## Dynamic User Filters

```python
class DynamicUserFilterView(generics.ListAPIView):
    """View with dynamic filtering based on URL parameters and user context"""
    serializer_class = TaskSerializer
    permission_classes = [IsAuthenticated]
    
    def get_queryset(self):
        user = self.request.user
        base_queryset = Task.objects.select_related(
            'project', 'assigned_to', 'created_by'
        )
        
        # Apply base user filtering
        accessible_projects = self.get_accessible_projects(user)
        queryset = base_queryset.filter(
            Q(project__in=accessible_projects) |
            Q(assigned_to=user) |
            Q(created_by=user)
        ).distinct()
        
        # Apply dynamic filters based on URL parameters
        return self.apply_dynamic_filters(queryset, user)
    
    def apply_dynamic_filters(self, queryset, user):
        """Apply filters based on query parameters and user context"""
        params = self.request.query_params
        
        # Filter by user relationship
        user_filter = params.get('user_filter')
        if user_filter == 'owned':
            # Tasks in projects owned by user
            queryset = queryset.filter(project__owner=user)
        elif user_filter == 'assigned':
            # Tasks assigned to user
            queryset = queryset.filter(assigned_to=user)
        elif user_filter == 'created':
            # Tasks created by user
            queryset = queryset.filter(created_by=user)
        elif user_filter == 'team':
            # Tasks from user's team members
            team_members = self.get_team_members(user)
            queryset = queryset.filter(
                Q(assigned_to__in=team_members) |
                Q(created_by__in=team_members)
            )
        
        # Filter by organization role
        role_filter = params.get('role_filter')
        if role_filter == 'admin_view':
            # Show all tasks in organizations where user is admin
            admin_orgs = user.membership_set.filter(
                role='admin'
            ).values_list('organization', flat=True)
            queryset = queryset.filter(project__organization__in=admin_orgs)
        
        # Filter by time scope
        time_filter = params.get('time_filter')
        if time_filter:
            queryset = self.apply_time_filter(queryset, time_filter)
        
        return queryset
    
    def get_team_members(self, user):
        """Get user's team members"""
        user_orgs = user.membership_set.values_list('organization', flat=True)
        return User.objects.filter(
            membership__organization__in=user_orgs
        ).exclude(id=user.id).distinct()
    
    def apply_time_filter(self, queryset, time_filter):
        """Apply time-based filtering"""
        from datetime import datetime, timedelta
        from django.utils import timezone
        
        now = timezone.now()
        
        if time_filter == 'today':
            start = now.replace(hour=0, minute=0, second=0, microsecond=0)
            return queryset.filter(created_at__gte=start)
        elif time_filter == 'week':
            start = now - timedelta(days=7)
            return queryset.filter(created_at__gte=start)
        elif time_filter == 'month':
            start = now - timedelta(days=30)
            return queryset.filter(created_at__gte=start)
        
        return queryset
    
    def get_accessible_projects(self, user):
        """Get projects accessible to user"""
        user_orgs = user.membership_set.values_list('organization', flat=True)
        
        return Project.objects.filter(
            Q(owner=user) |
            Q(collaborators=user) |
            Q(is_public=True, organization__in=user_orgs)
        ).distinct()
```

## Permission-Based Filtering

```python
# permissions.py
from rest_framework import permissions

class OrganizationMemberPermission(permissions.BasePermission):
    """Permission that also provides filtering context"""
    
    def has_permission(self, request, view):
        if not request.user.is_authenticated:
            return False
        
        # Store user's accessible organizations in view for filtering
        view.user_organizations = request.user.membership_set.values_list(
            'organization', flat=True
        )
        
        return True
    
    def has_object_permission(self, request, view, obj):
        if hasattr(obj, 'organization'):
            return obj.organization.id in view.user_organizations
        elif hasattr(obj, 'project'):
            return obj.project.organization.id in view.user_organizations
        
        return False

class PermissionFilteredViewSet(viewsets.ModelViewSet):
    permission_classes = [OrganizationMemberPermission]
    
    def get_queryset(self):
        """Use permissions to determine filtering"""
        queryset = super().get_queryset()
        
        # Use organizations from permission check
        if hasattr(self, 'user_organizations'):
            if hasattr(queryset.model, 'organization'):
                return queryset.filter(organization__in=self.user_organizations)
        
        return queryset
```

## URL Examples

With user-based filtering, your API provides these filtered endpoints:

```
# Basic user filtering
GET /api/projects/              # Projects user has access to
GET /api/tasks/                 # Tasks user can see
GET /api/my-tasks/              # Tasks assigned to/created by user

# Dynamic user filtering
GET /api/tasks/?user_filter=assigned      # Only assigned tasks
GET /api/tasks/?user_filter=owned         # Tasks in owned projects
GET /api/tasks/?user_filter=team          # Team tasks
GET /api/tasks/?role_filter=admin_view    # Admin view

# Time-based user filtering
GET /api/tasks/?time_filter=today         # Today's tasks
GET /api/tasks/?time_filter=week          # This week's tasks

# Organization-scoped filtering
GET /api/projects/?org=123                # Projects in specific org (if user has access)
```

## Notes

### Security Considerations

- **Never Trust Client Input**: Always validate user access server-side
- **Defense in Depth**: Combine filtering with permissions and object-level checks
- **Audit Trails**: Log access to sensitive data
- **Information Leakage**: Avoid exposing existence of data user can't access

### Performance Tips

1. **Database Indexes**: Add indexes on user relationship fields
2. **Prefetch Related**: Use `prefetch_related()` for many-to-many relationships
3. **Select Related**: Use `select_related()` for foreign key relationships
4. **Subquery Optimization**: Consider using `Exists()` for complex filters
5. **Caching**: Cache user organization/role data

### Common Patterns

1. **Ownership**: Filter by `owner`, `created_by`, `assigned_to`
2. **Membership**: Filter by organization/team membership
3. **Role-Based**: Different access levels based on user roles
4. **Multi-Tenant**: Scope data by tenant/organization
5. **Hierarchical**: Respect organization/department hierarchies

### Error Handling

- Return empty querysets for unauthorized access
- Use appropriate HTTP status codes (404 vs 403)
- Provide meaningful error messages
- Log security violations for monitoring

This recipe ensures users only see data they're authorized to access while maintaining clean, efficient API design.
