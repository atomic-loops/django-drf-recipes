# Custom Permissions

## Problem

You need to implement more granular access control in your Django REST Framework API beyond the built-in permissions, such as allowing users to only access their own data or implementing role-based permissions.

## Solution

Create custom permission classes by extending DRF's `BasePermission` class and implementing custom logic for object-level and view-level permissions.

## Code

### 1. Basic Permission Structure

DRF permissions have two main methods you can override:

- `has_permission(self, request, view)`: View-level permission check
- `has_object_permission(self, request, view, obj)`: Object-level permission check

### 2. IsOwner Permission

Let's create a permission to only allow users to access their own resources:

```python
# permissions.py
from rest_framework import permissions

class IsOwner(permissions.BasePermission):
    """
    Custom permission to only allow owners of an object to access it.
    """
    def has_object_permission(self, request, view, obj):
        # Read permissions are allowed to any request,
        # so we'll always allow GET, HEAD or OPTIONS requests.
        if request.method in permissions.SAFE_METHODS:
            return True

        # Write permissions are only allowed to the owner of the object.
        return obj.owner == request.user
```

Apply the permission to a view:

```python
# views.py
from rest_framework import viewsets
from .models import Post
from .serializers import PostSerializer
from .permissions import IsOwner

class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    
    def get_permissions(self):
        """
        Instantiates and returns the list of permissions that this view requires.
        """
        if self.action in ['update', 'partial_update', 'destroy']:
            permission_classes = [IsOwner]
        else:
            permission_classes = [permissions.IsAuthenticated]
        return [permission() for permission in permission_classes]
    
    def perform_create(self, serializer):
        serializer.save(owner=self.request.user)
```

### 3. IsAdminOrReadOnly Permission

A permission that allows only admin users to make changes:

```python
# permissions.py
from rest_framework import permissions

class IsAdminOrReadOnly(permissions.BasePermission):
    """
    Custom permission to only allow admins to edit but anyone to view.
    """
    def has_permission(self, request, view):
        # Read permissions are allowed to any request
        if request.method in permissions.SAFE_METHODS:
            return True
            
        # Write permissions are only allowed to admin users
        return request.user and request.user.is_staff
```

### 4. IsAdminOrOwnerOrReadOnly Permission

A permission that combines admin access with owner-specific access:

```python
# permissions.py
from rest_framework import permissions

class IsAdminOrOwnerOrReadOnly(permissions.BasePermission):
    """
    Custom permission:
    - Allow read-only access for any authenticated user
    - Allow write access for admins
    - Allow write access for the owner of an object
    """
    def has_permission(self, request, view):
        # Read permissions are allowed to authenticated users
        if request.method in permissions.SAFE_METHODS:
            return request.user and request.user.is_authenticated
            
        # Write permissions are allowed to authenticated users
        # (object-level permissions will be checked)
        return request.user and request.user.is_authenticated
        
    def has_object_permission(self, request, view, obj):
        # Read permissions are allowed to authenticated users
        if request.method in permissions.SAFE_METHODS:
            return request.user and request.user.is_authenticated
            
        # Write permissions are allowed to admins
        if request.user and request.user.is_staff:
            return True
            
        # Write permissions are allowed to the owner of the object
        return hasattr(obj, 'owner') and obj.owner == request.user
```

### 5. Combining Multiple Permissions

You can apply multiple permissions to a view:

```python
# views.py
from rest_framework import viewsets
from rest_framework.permissions import IsAuthenticated
from .models import Post
from .serializers import PostSerializer
from .permissions import IsOwner

class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    permission_classes = [IsAuthenticated, IsOwner]  # User must pass BOTH permissions
```

### 6. DjangoModelPermissions

DRF also includes permission classes that use Django's model permissions:

```python
# views.py
from rest_framework import viewsets
from rest_framework.permissions import DjangoModelPermissions
from .models import Post
from .serializers import PostSerializer

class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    permission_classes = [DjangoModelPermissions]
```

With this permission, the view will check user permissions against the model:
- `add` permission for POST requests
- `change` permission for PUT/PATCH requests
- `delete` permission for DELETE requests

### 7. Custom Field-Level Permissions

For field-level permissions, customize your serializer:

```python
# serializers.py
from rest_framework import serializers
from .models import Profile

class ProfileSerializer(serializers.ModelSerializer):
    class Meta:
        model = Profile
        fields = ['id', 'user', 'bio', 'birth_date', 'location', 'premium_status']
        
    def to_representation(self, instance):
        """
        Remove premium_status field for non-staff users.
        """
        ret = super().to_representation(instance)
        user = self.context['request'].user
        
        if not user.is_staff:
            ret.pop('premium_status', None)
            
        return ret
        
    def validate(self, data):
        """
        Check that only staff can update premium_status.
        """
        user = self.context['request'].user
        
        if 'premium_status' in data and not user.is_staff:
            raise serializers.ValidationError({
                'premium_status': 'Only staff members can update premium status.'
            })
            
        return data
```

### 8. Customizing DjangoObjectPermissions

You can extend Django's object permissions system:

```python
# permissions.py
from rest_framework import permissions

class CustomObjectPermissions(permissions.DjangoObjectPermissions):
    """
    Similar to DjangoObjectPermissions, but allows read permissions to any 
    authenticated user.
    """
    def has_permission(self, request, view):
        # Read permissions are allowed to any authenticated user
        if request.method in permissions.SAFE_METHODS:
            return request.user and request.user.is_authenticated
            
        # Write permissions require object permissions
        return super().has_permission(request, view)
```

## Notes

1. **Permission Evaluation**:
   - `has_permission()` is always checked first
   - `has_object_permission()` is only checked if `has_permission()` returns `True`
   - For list views, only `has_permission()` is checked

2. **Working with DRF Generic Views**:
   - `GenericAPIView` automatically calls `check_object_permissions()`
   - For custom views, call `self.check_object_permissions(request, obj)` manually

3. **Combining Permissions**:
   - When using a list of permission classes, ALL must return `True`
   - To implement OR logic, create a custom permission that combines conditions

4. **Performance Considerations**:
   - Complex permission checks can impact API performance
   - Consider optimizing database queries in permission checks
   - For list views, filter the queryset instead of checking permissions on each object

5. **Testing Permissions**:
   - Use APITestCase to thoroughly test permission logic
   - Test both authenticated and unauthenticated access
   - Test different user roles (regular users, admins, etc.)
   - Test object-level permissions with different object owners

6. **Common Patterns**:
   - Allow read-only access to authenticated users
   - Allow full access to object owners
   - Allow full access to admins
   - Combine role-based and object-based permissions

7. **Global vs. View-Specific Permissions**:
   - Set default permissions in `REST_FRAMEWORK` settings
   - Override on a per-view basis using `permission_classes`
   - Use `get_permissions()` for dynamic permission assignment
