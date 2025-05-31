# File Upload

## Problem
How do you handle file uploads (e.g., images, documents) via a Django REST Framework API?

## Solution
Use DRF's `FileField` or `ImageField` in a serializer, and a view to handle multipart form data uploads.

## Code

### models.py
```python
from django.db import models

class UploadedFile(models.Model):
    file = models.FileField(upload_to='uploads/')
    uploaded_at = models.DateTimeField(auto_now_add=True)
```

### serializers.py
```python
from rest_framework import serializers
from .models import UploadedFile

class UploadedFileSerializer(serializers.ModelSerializer):
    class Meta:
        model = UploadedFile
        fields = ['id', 'file', 'uploaded_at']
```

### views.py
```python
from rest_framework import generics, permissions
from .models import UploadedFile
from .serializers import UploadedFileSerializer

class FileUploadView(generics.CreateAPIView):
    queryset = UploadedFile.objects.all()
    serializer_class = UploadedFileSerializer
    permission_classes = [permissions.IsAuthenticated]
```

### urls.py
```python
from django.urls import path
from .views import FileUploadView

urlpatterns = [
    path('upload/', FileUploadView.as_view(), name='file-upload'),
]
```

## Notes
- Ensure your Django settings are configured for media file handling (`MEDIA_URL`, `MEDIA_ROOT`).
- Use `ImageField` for image uploads and install Pillow if needed.
- For large files or production, consider using cloud storage backends (e.g., S3) with `django-storages`.
- Always validate file types and sizes in production for security. 