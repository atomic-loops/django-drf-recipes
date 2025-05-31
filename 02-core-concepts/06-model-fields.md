# Django Model Fields: Types & Usage

Understanding Django model fields is essential for designing robust APIs and databases. This guide covers the most common field types, their options, and how they map to Django REST Framework (DRF) serializers.

## Common Field Types

| Field Type         | Description                                 | Example Usage                          |
|--------------------|---------------------------------------------|----------------------------------------|
| `CharField`        | Text field with a max length                | `name = models.CharField(max_length=50)`|
| `TextField`        | Large text field                            | `bio = models.TextField()`             |
| `IntegerField`     | Integer values                              | `age = models.IntegerField()`          |
| `FloatField`       | Floating point numbers                      | `price = models.FloatField()`          |
| `BooleanField`     | True/False values                           | `is_active = models.BooleanField()`    |
| `DateTimeField`    | Date and time                               | `created = models.DateTimeField(auto_now_add=True)` |
| `DateField`        | Date only                                   | `birth_date = models.DateField()`      |
| `TimeField`        | Time only                                   | `start_time = models.TimeField()`      |
| `EmailField`       | Email addresses                             | `email = models.EmailField()`          |
| `URLField`         | URLs                                        | `website = models.URLField()`          |
| `FileField`        | File uploads                                | `document = models.FileField(upload_to='docs/')` |
| `ImageField`       | Image uploads                               | `avatar = models.ImageField(upload_to='avatars/')` |
| `ForeignKey`       | Many-to-one relationship                    | `author = models.ForeignKey(User, on_delete=models.CASCADE)` |
| `ManyToManyField`  | Many-to-many relationship                   | `tags = models.ManyToManyField(Tag)`   |
| `OneToOneField`    | One-to-one relationship                     | `profile = models.OneToOneField(Profile, on_delete=models.CASCADE)` |

## Field Options
- `null`: If True, Django will store empty values as NULL in the database.
- `blank`: If True, the field is allowed to be blank in forms.
- `default`: Default value for the field.
- `choices`: Restrict the field to a set of choices.
- `unique`: If True, this field must be unique throughout the table.

**Example:**
```python
class Product(models.Model):
    STATUS_CHOICES = [
        ('draft', 'Draft'),
        ('published', 'Published'),
    ]
    name = models.CharField(max_length=100, unique=True)
    price = models.DecimalField(max_digits=8, decimal_places=2)
    status = models.CharField(max_length=10, choices=STATUS_CHOICES, default='draft')
    created = models.DateTimeField(auto_now_add=True)
```

## Custom Model Fields
You can create custom fields by subclassing `models.Field` for advanced use cases.

## Model Fields & DRF Serializers
Django REST Framework automatically maps model fields to serializer fields:
- `CharField` → `serializers.CharField`
- `IntegerField` → `serializers.IntegerField`
- `ForeignKey` → `serializers.PrimaryKeyRelatedField` (or nested serializer)

**Example:**
```python
class ProductSerializer(serializers.ModelSerializer):
    class Meta:
        model = Product
        fields = '__all__'
```

## Best Practices
- Use the most specific field type possible (e.g., `EmailField` for emails).
- Use `choices` for enumerated values.
- Set `null` and `blank` appropriately for your use case.
- Use related fields (`ForeignKey`, `ManyToManyField`) to model relationships.

---

For more details, see the [Django model field documentation](https://docs.djangoproject.com/en/stable/ref/models/fields/). 