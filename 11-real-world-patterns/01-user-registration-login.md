# User Registration & Login

## Problem
How do you implement secure user registration and login endpoints in a Django REST Framework API?

## Solution
Use Django's built-in User model with DRF serializers and views to create registration and login endpoints. Use token authentication for login.

## Code

### serializers.py
```python
from django.contrib.auth.models import User
from rest_framework import serializers
from rest_framework.authtoken.models import Token

class RegisterSerializer(serializers.ModelSerializer):
    password = serializers.CharField(write_only=True)

    class Meta:
        model = User
        fields = ('username', 'email', 'password')

    def create(self, validated_data):
        user = User.objects.create_user(
            username=validated_data['username'],
            email=validated_data['email'],
            password=validated_data['password']
        )
        Token.objects.create(user=user)
        return user

class LoginSerializer(serializers.Serializer):
    username = serializers.CharField()
    password = serializers.CharField(write_only=True)
```

### views.py
```python
from rest_framework import generics, status
from rest_framework.response import Response
from rest_framework.authtoken.models import Token
from django.contrib.auth import authenticate
from .serializers import RegisterSerializer, LoginSerializer

class RegisterView(generics.CreateAPIView):
    serializer_class = RegisterSerializer

class LoginView(generics.GenericAPIView):
    serializer_class = LoginSerializer

    def post(self, request, *args, **kwargs):
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        user = authenticate(
            username=serializer.validated_data['username'],
            password=serializer.validated_data['password']
        )
        if user:
            token, created = Token.objects.get_or_create(user=user)
            return Response({'token': token.key})
        return Response({'error': 'Invalid Credentials'}, status=status.HTTP_400_BAD_REQUEST)
```

### urls.py
```python
from django.urls import path
from .views import RegisterView, LoginView

urlpatterns = [
    path('register/', RegisterView.as_view(), name='register'),
    path('login/', LoginView.as_view(), name='login'),
]
```

## Notes
- Use HTTPS in production to protect credentials.
- Consider using packages like `djoser` or `dj-rest-auth` for more advanced authentication flows.
- Token authentication is simple but for production, JWT (with `djangorestframework-simplejwt`) is recommended for stateless APIs. 