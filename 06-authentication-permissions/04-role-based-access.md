# Role-Based Access Control

## Problem

You need to implement a more sophisticated permission system beyond basic user authentication, where users have different roles that determine what actions they can perform in your API.

## Solution

Implement Role-Based Access Control (RBAC) in Django REST Framework by leveraging Django's built-in groups and permissions system along with custom permission classes.

## Code

### 1. Basic Role Setup with Django Groups

First, set up the role structure using Django's built-in groups:

```python
# roles.py
from django.contrib.auth.models import Group, Permission
from django.contrib.contenttypes.models import ContentType
from django.db import models

def create_base_roles():
    """
    Create basic roles for the application.
    """
    # Create groups if they don't exist
    admin_group, _ = Group.objects.get_or_create(name='Administrators')
    editor_group, _ = Group.objects.get_or_create(name='Editors')
    viewer_group, _ = Group.objects.get_or_create(name='Viewers')
    
    # Get content types for the models we want to assign permissions to
    book_content_type = ContentType.objects.get(app_label='library', model='book')
    
    # Get or create permissions
    view_book = Permission.objects.get(content_type=book_content_type, codename='view_book')
    add_book = Permission.objects.get(content_type=book_content_type, codename='add_book')
    change_book = Permission.objects.get(content_type=book_content_type, codename='change_book')
    delete_book = Permission.objects.get(content_type=book_content_type, codename='delete_book')
    
    # Assign permissions to groups
    # Administrators can do everything
    admin_group.permissions.add(view_book, add_book, change_book, delete_book)
    
    # Editors can view, add, and change but not delete
    editor_group.permissions.add(view_book, add_book, change_book)
    
    # Viewers can only view
    viewer_group.permissions.add(view_book)
```

You can call this function in a migration or management command to set up your roles:

```python
# management/commands/setup_roles.py
from django.core.management.base import BaseCommand
from myapp.roles import create_base_roles

class Command(BaseCommand):
    help = 'Sets up the default roles and permissions'

    def handle(self, *args, **options):
        create_base_roles()
        self.stdout.write(self.style.SUCCESS('Successfully created roles'))
```

### 2. Using Django's Default Permission Classes

Django REST Framework provides a permission class that works with Django's permission system:

```python
# views.py
from rest_framework import viewsets
from rest_framework.permissions import DjangoModelPermissions
from .models import Book
from .serializers import BookSerializer

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    permission_classes = [DjangoModelPermissions]
```

This will automatically enforce:
- Users with `add_book` can POST
- Users with `change_book` can PUT/PATCH
- Users with `delete_book` can DELETE
- Users with `view_book` can GET (or any user with `add`, `change`, or `delete` permissions)

### 3. Custom Role-Based Permission

For more fine-grained control, create a custom permission class:

```python
# permissions.py
from rest_framework import permissions

class RoleBasedPermission(permissions.BasePermission):
    """
    Permission class that checks for specific roles.
    """
    def has_permission(self, request, view):
        # Check if user is authenticated
        if not request.user or not request.user.is_authenticated:
            return False
        
        # Superusers can do anything
        if request.user.is_superuser:
            return True
        
        # Check user roles (groups)
        user_groups = request.user.groups.values_list('name', flat=True)
        
        # Define role-based permissions
        if view.action == 'list' or view.action == 'retrieve':
            # Anyone with any role can view
            return bool(user_groups)
        elif view.action == 'create':
            # Only Editors and Administrators can create
            return 'Editors' in user_groups or 'Administrators' in user_groups
        elif view.action in ['update', 'partial_update']:
            # Only Editors and Administrators can update
            return 'Editors' in user_groups or 'Administrators' in user_groups
        elif view.action == 'destroy':
            # Only Administrators can delete
            return 'Administrators' in user_groups
        
        # Default deny
        return False
```

Use this permission in your viewset:

```python
# views.py
from rest_framework import viewsets
from .models import Book
from .serializers import BookSerializer
from .permissions import RoleBasedPermission

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    permission_classes = [RoleBasedPermission]
```

### 4. Object-Level Role-Based Permissions

For object-level permissions based on roles:

```python
# permissions.py
from rest_framework import permissions

class RoleBasedObjectPermission(permissions.BasePermission):
    """
    Object-level permission that checks for specific roles.
    """
    def has_permission(self, request, view):
        # Check if user is authenticated
        return request.user and request.user.is_authenticated
    
    def has_object_permission(self, request, view, obj):
        # Superusers can do anything
        if request.user.is_superuser:
            return True
        
        # Check user roles (groups)
        user_groups = request.user.groups.values_list('name', flat=True)
        
        # Define role-based object permissions
        if view.action == 'retrieve':
            # Anyone with any role can view
            return bool(user_groups)
        elif view.action in ['update', 'partial_update']:
            # Editors can update only their own objects
            # Administrators can update any object
            if 'Administrators' in user_groups:
                return True
            elif 'Editors' in user_groups:
                return obj.created_by == request.user
            return False
        elif view.action == 'destroy':
            # Only Administrators can delete any object
            # Editors can delete only their own objects
            if 'Administrators' in user_groups:
                return True
            elif 'Editors' in user_groups:
                return obj.created_by == request.user
            return False
        
        # Default allow for safe methods, deny for others
        return request.method in permissions.SAFE_METHODS
```

### 5. Custom Role Model

For more complex role requirements, you might need a custom role model:

```python
# models.py
from django.db import models
from django.contrib.auth.models import User

class Role(models.Model):
    """
    Custom role model for more complex role definitions.
    """
    name = models.CharField(max_length=100, unique=True)
    description = models.TextField(blank=True)
    
    # Define permissions as choices
    class Permissions(models.TextChoices):
        VIEW = 'view', 'View'
        CREATE = 'create', 'Create'
        EDIT = 'edit', 'Edit'
        DELETE = 'delete', 'Delete'
        APPROVE = 'approve', 'Approve'
        PUBLISH = 'publish', 'Publish'
    
    # Many-to-many relationship with permissions
    permissions = models.JSONField(default=list)
    
    def __str__(self):
        return self.name

class UserRole(models.Model):
    """
    Association between User and Role.
    """
    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name='user_roles')
    role = models.ForeignKey(Role, on_delete=models.CASCADE)
    
    class Meta:
        unique_together = ('user', 'role')
    
    def __str__(self):
        return f"{self.user.username} - {self.role.name}"
```

Then create a permission class that uses this custom role model:

```python
# permissions.py
from rest_framework import permissions
from .models import UserRole

class CustomRolePermission(permissions.BasePermission):
    """
    Permission class using custom role model.
    """
    def has_permission(self, request, view):
        # Check if user is authenticated
        if not request.user or not request.user.is_authenticated:
            return False
        
        # Superusers can do anything
        if request.user.is_superuser:
            return True
        
        # Get user roles
        user_roles = UserRole.objects.filter(user=request.user).select_related('role')
        
        # Map actions to permissions
        action_to_permission = {
            'list': 'view',
            'retrieve': 'view',
            'create': 'create',
            'update': 'edit',
            'partial_update': 'edit',
            'destroy': 'delete',
        }
        
        # Custom actions
        if view.action == 'approve':
            required_permission = 'approve'
        elif view.action == 'publish':
            required_permission = 'publish'
        else:
            required_permission = action_to_permission.get(view.action)
        
        # Check if user has any role with the required permission
        for user_role in user_roles:
            if required_permission in user_role.role.permissions:
                return True
        
        # Default deny
        return False
```

### 6. Role-Based Access with API Scopes

For API-level scopes based on roles:

```python
# models.py
from django.db import models
from django.contrib.auth.models import User

class ApiScope(models.Model):
    """
    API scope for role-based access control.
    """
    name = models.CharField(max_length=100, unique=True)
    description = models.TextField(blank=True)
    
    def __str__(self):
        return self.name

class Role(models.Model):
    """
    Role model with API scopes.
    """
    name = models.CharField(max_length=100, unique=True)
    description = models.TextField(blank=True)
    scopes = models.ManyToManyField(ApiScope)
    
    def __str__(self):
        return self.name

class UserRole(models.Model):
    """
    Association between User and Role.
    """
    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name='user_roles')
    role = models.ForeignKey(Role, on_delete=models.CASCADE)
    
    class Meta:
        unique_together = ('user', 'role')
```

Then create a permission class that checks API scopes:

```python
# permissions.py
from rest_framework import permissions
from .models import UserRole, ApiScope

class ScopeBasedPermission(permissions.BasePermission):
    """
    Permission class that checks API scopes.
    """
    def has_permission(self, request, view):
        # Check if user is authenticated
        if not request.user or not request.user.is_authenticated:
            return False
        
        # Superusers can do anything
        if request.user.is_superuser:
            return True
        
        # Get required scope for this view
        required_scope = getattr(view, 'required_scope', None)
        if not required_scope:
            return True  # No scope required
        
        # Get user roles and their scopes
        user_roles = UserRole.objects.filter(user=request.user).select_related('role')
        user_scopes = set()
        
        for user_role in user_roles:
            role_scopes = user_role.role.scopes.values_list('name', flat=True)
            user_scopes.update(role_scopes)
        
        # Check if user has the required scope
        return required_scope in user_scopes
```

Use this permission in your viewset:

```python
# views.py
from rest_framework import viewsets
from .models import Book
from .serializers import BookSerializer
from .permissions import ScopeBasedPermission

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    permission_classes = [ScopeBasedPermission]
    required_scope = 'books:read'  # Scope required for this view
    
    def get_required_scope(self):
        """
        Get the required scope based on the action.
        """
        action_to_scope = {
            'list': 'books:read',
            'retrieve': 'books:read',
            'create': 'books:write',
            'update': 'books:write',
            'partial_update': 'books:write',
            'destroy': 'books:delete',
        }
        return action_to_scope.get(self.action, 'books:read')
```

### 7. Role Hierarchy

Implement a role hierarchy where higher roles inherit permissions from lower roles:

```python
# models.py
from django.db import models
from django.contrib.auth.models import User

class Role(models.Model):
    """
    Role model with hierarchy.
    """
    name = models.CharField(max_length=100, unique=True)
    description = models.TextField(blank=True)
    parent = models.ForeignKey('self', null=True, blank=True, on_delete=models.SET_NULL, related_name='children')
    permissions = models.JSONField(default=list)
    
    def get_all_permissions(self):
        """
        Get all permissions, including inherited from parent roles.
        """
        all_permissions = set(self.permissions)
        
        # Recursively get permissions from parent roles
        if self.parent:
            all_permissions.update(self.parent.get_all_permissions())
        
        return all_permissions
    
    def __str__(self):
        return self.name
```

### 8. Access Control Lists (ACLs)

For very fine-grained control, implement ACLs:

```python
# models.py
from django.db import models
from django.contrib.auth.models import User
from django.contrib.contenttypes.fields import GenericForeignKey
from django.contrib.contenttypes.models import ContentType

class AccessControlList(models.Model):
    """
    Access Control List for very fine-grained permissions.
    """
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    
    # Generic relationship to any model
    content_type = models.ForeignKey(ContentType, on_delete=models.CASCADE)
    object_id = models.PositiveIntegerField()
    content_object = GenericForeignKey('content_type', 'object_id')
    
    # Permission types
    can_view = models.BooleanField(default=False)
    can_edit = models.BooleanField(default=False)
    can_delete = models.BooleanField(default=False)
    
    class Meta:
        unique_together = ('user', 'content_type', 'object_id')
```

Create a permission class that uses these ACLs:

```python
# permissions.py
from rest_framework import permissions
from django.contrib.contenttypes.models import ContentType
from .models import AccessControlList

class ACLPermission(permissions.BasePermission):
    """
    Permission class that checks Access Control Lists.
    """
    def has_object_permission(self, request, view, obj):
        # Superusers can do anything
        if request.user.is_superuser:
            return True
        
        # Get content type of the object
        content_type = ContentType.objects.get_for_model(obj)
        
        try:
            # Get ACL for this user and object
            acl = AccessControlList.objects.get(
                user=request.user,
                content_type=content_type,
                object_id=obj.id
            )
            
            # Check permissions based on request method
            if request.method in permissions.SAFE_METHODS:
                return acl.can_view
            elif request.method in ['PUT', 'PATCH']:
                return acl.can_edit
            elif request.method == 'DELETE':
                return acl.can_delete
        except AccessControlList.DoesNotExist:
            # No ACL found, default to deny
            return False
        
        return False
```

### 9. Combining RBAC with JWT

You can include role information in JWT tokens:

```python
# utils.py
from rest_framework_simplejwt.serializers import TokenObtainPairSerializer
from rest_framework_simplejwt.views import TokenObtainPairView

class RoleAwareTokenSerializer(TokenObtainPairSerializer):
    """
    Custom token serializer that includes user roles in the token.
    """
    @classmethod
    def get_token(cls, user):
        token = super().get_token(user)
        
        # Add user roles to the token
        roles = user.groups.values_list('name', flat=True)
        token['roles'] = list(roles)
        
        return token

class RoleAwareTokenView(TokenObtainPairView):
    """
    Custom token view that uses the role-aware token serializer.
    """
    serializer_class = RoleAwareTokenSerializer
```

Register this view in your URLs:

```python
# urls.py
from django.urls import path
from .utils import RoleAwareTokenView

urlpatterns = [
    # ... other URL patterns
    path('api/token/', RoleAwareTokenView.as_view(), name='token_obtain_pair'),
]
```

### 10. Role Assignment API

Create API endpoints for role management:

```python
# views.py
from rest_framework import viewsets, status
from rest_framework.decorators import action
from rest_framework.response import Response
from django.contrib.auth.models import User, Group
from .serializers import UserSerializer, GroupSerializer
from .permissions import IsAdminUser

class UserViewSet(viewsets.ModelViewSet):
    queryset = User.objects.all()
    serializer_class = UserSerializer
    permission_classes = [IsAdminUser]
    
    @action(detail=True, methods=['post'])
    def assign_role(self, request, pk=None):
        user = self.get_object()
        role_id = request.data.get('role_id')
        
        try:
            role = Group.objects.get(id=role_id)
            user.groups.add(role)
            return Response({'status': f"Role '{role.name}' assigned to {user.username}"})
        except Group.DoesNotExist:
            return Response(
                {'error': 'Role not found'},
                status=status.HTTP_404_NOT_FOUND
            )
    
    @action(detail=True, methods=['post'])
    def remove_role(self, request, pk=None):
        user = self.get_object()
        role_id = request.data.get('role_id')
        
        try:
            role = Group.objects.get(id=role_id)
            if role in user.groups.all():
                user.groups.remove(role)
                return Response({'status': f"Role '{role.name}' removed from {user.username}"})
            else:
                return Response(
                    {'error': f"User does not have role '{role.name}'"},
                    status=status.HTTP_400_BAD_REQUEST
                )
        except Group.DoesNotExist:
            return Response(
                {'error': 'Role not found'},
                status=status.HTTP_404_NOT_FOUND
            )
```

## Notes

1. **Role Design Considerations**:
   - Keep the role structure simple, with a small number of well-defined roles
   - Avoid creating too many specialized roles (role explosion)
   - Design roles based on job functions, not individuals
   - Consider using role hierarchies for large organizations

2. **Django Groups vs Custom Roles**:
   - Django Groups are sufficient for simple role-based access control
   - Custom role models offer more flexibility for complex requirements
   - Django Groups are easier to use with Django admin
   - Custom roles can support hierarchies and fine-grained permissions

3. **Performance Considerations**:
   - Cache permission checks when possible
   - Use `select_related` and `prefetch_related` to reduce database queries
   - Consider storing role information in JWT tokens to reduce database lookups
   - Test performance with a realistic number of roles and permissions

4. **Security Best Practices**:
   - Implement the principle of least privilege
   - Regularly audit role assignments
   - Log permission checks and access denials
   - Consider time-based or context-based restrictions for sensitive operations

5. **Testing RBAC**:
   - Test each role against all endpoints
   - Test role transitions (adding/removing roles)
   - Test hierarchical permissions
   - Test edge cases (no roles, multiple roles, etc.)

6. **Additional Features**:
   - Temporary role assignments
   - Delegated administration (role managers)
   - Context-based roles (roles that apply only in certain contexts)
   - Approval workflows for role assignments
   - Role audit logs

7. **Integration with Other Systems**:
   - Consider integrating with enterprise identity providers
   - Support for SAML or OIDC for role information
   - Synchronizing roles with external systems
