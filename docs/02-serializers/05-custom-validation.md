# Custom Validation

## Problem

You need to implement custom validation logic for your API endpoints that goes beyond the basic field validations provided by Django REST Framework.

## Solution

Use DRF's serializer validation hooks to implement field-level, object-level, and custom validators for complex validation scenarios.

## Code

### 1. Field-Level Validation

Field-level validation is done by adding `validate_<field_name>` methods to your serializer:

```python
# serializers.py
from rest_framework import serializers
from .models import Book
import datetime

class BookSerializer(serializers.ModelSerializer):
    class Meta:
        model = Book
        fields = ['id', 'title', 'author', 'publication_date', 'isbn', 'price']
    
    def validate_title(self, value):
        """
        Check that the title isn't too short.
        """
        if len(value) < 3:
            raise serializers.ValidationError("Title must be at least 3 characters long.")
        return value
    
    def validate_publication_date(self, value):
        """
        Check that the publication date is not in the future.
        """
        if value > datetime.date.today():
            raise serializers.ValidationError("Publication date cannot be in the future.")
        return value
    
    def validate_isbn(self, value):
        """
        Check that the ISBN is valid.
        """
        # Remove hyphens and spaces
        isbn = value.replace('-', '').replace(' ', '')
        
        # Check if it's a valid ISBN-10 or ISBN-13
        if len(isbn) == 10:
            # ISBN-10 validation
            if not isbn[:-1].isdigit() or not (isbn[-1].isdigit() or isbn[-1] == 'X'):
                raise serializers.ValidationError("Invalid ISBN-10 format.")
        elif len(isbn) == 13:
            # ISBN-13 validation
            if not isbn.isdigit():
                raise serializers.ValidationError("Invalid ISBN-13 format.")
        else:
            raise serializers.ValidationError("ISBN must be 10 or 13 characters long.")
        
        return value
    
    def validate_price(self, value):
        """
        Check that the price is positive.
        """
        if value <= 0:
            raise serializers.ValidationError("Price must be greater than zero.")
        return value
```

### 2. Object-Level Validation

Object-level validation allows you to validate based on multiple fields using the `validate()` method:

```python
# serializers.py
from rest_framework import serializers
from .models import Event
import datetime

class EventSerializer(serializers.ModelSerializer):
    class Meta:
        model = Event
        fields = ['id', 'title', 'start_time', 'end_time', 'location']
    
    def validate(self, data):
        """
        Check that the start time is before the end time.
        """
        if 'start_time' in data and 'end_time' in data:
            if data['start_time'] >= data['end_time']:
                raise serializers.ValidationError({
                    "end_time": "End time must be after start time."
                })
        
        # Validate the event doesn't last more than a week
        if 'start_time' in data and 'end_time' in data:
            duration = data['end_time'] - data['start_time']
            if duration.days > 7:
                raise serializers.ValidationError({
                    "non_field_errors": ["Event duration cannot exceed 7 days."]
                })
        
        return data
```

### 3. Validators Array

You can also use a validators array in the field definition:

```python
# serializers.py
from rest_framework import serializers
from .models import User
import re

def validate_password_strength(value):
    """
    Validate that the password meets strength requirements.
    """
    if len(value) < 8:
        raise serializers.ValidationError("Password must be at least 8 characters long.")
    
    if not any(char.isdigit() for char in value):
        raise serializers.ValidationError("Password must contain at least one digit.")
    
    if not any(char.isupper() for char in value):
        raise serializers.ValidationError("Password must contain at least one uppercase letter.")
    
    if not any(char.islower() for char in value):
        raise serializers.ValidationError("Password must contain at least one lowercase letter.")
    
    if not any(char in "!@#$%^&*()-_=+[]{}|;:,.<>?/" for char in value):
        raise serializers.ValidationError("Password must contain at least one special character.")
    
    return value

class UserSerializer(serializers.ModelSerializer):
    password = serializers.CharField(
        write_only=True,
        validators=[validate_password_strength]
    )
    
    class Meta:
        model = User
        fields = ['id', 'username', 'email', 'password']
    
    def create(self, validated_data):
        user = User.objects.create_user(**validated_data)
        return user
```

### 4. Using Built-in Validators

DRF provides several built-in validators you can use:

```python
# serializers.py
from rest_framework import serializers
from rest_framework.validators import UniqueValidator, UniqueTogetherValidator
from .models import Product, Category

class CategorySerializer(serializers.ModelSerializer):
    name = serializers.CharField(
        validators=[
            UniqueValidator(
                queryset=Category.objects.all(),
                message="A category with this name already exists."
            )
        ]
    )
    
    class Meta:
        model = Category
        fields = ['id', 'name', 'description']

class ProductSerializer(serializers.ModelSerializer):
    class Meta:
        model = Product
        fields = ['id', 'name', 'sku', 'category', 'price']
        validators = [
            UniqueTogetherValidator(
                queryset=Product.objects.all(),
                fields=['name', 'category'],
                message="A product with this name already exists in this category."
            )
        ]
```

### 5. Database-Level Validation

Sometimes you want to validate against the database:

```python
# serializers.py
from rest_framework import serializers
from .models import Reservation, Room

class ReservationSerializer(serializers.ModelSerializer):
    class Meta:
        model = Reservation
        fields = ['id', 'room', 'guest_name', 'check_in', 'check_out']
    
    def validate(self, data):
        """
        Check that the room is available for the specified dates.
        """
        room = data.get('room')
        check_in = data.get('check_in')
        check_out = data.get('check_out')
        
        # Validate that check-in is before check-out
        if check_in >= check_out:
            raise serializers.ValidationError({
                "check_out": "Check-out must be after check-in."
            })
        
        # Validate room availability
        overlapping_reservations = Reservation.objects.filter(
            room=room,
            check_out__gt=check_in,
            check_in__lt=check_out
        )
        
        # Exclude current instance when updating
        if self.instance:
            overlapping_reservations = overlapping_reservations.exclude(pk=self.instance.pk)
        
        if overlapping_reservations.exists():
            raise serializers.ValidationError({
                "non_field_errors": ["This room is not available for the selected dates."]
            })
        
        return data
```

### 6. Conditional Validation

You might need to apply validation rules conditionally:

```python
# serializers.py
from rest_framework import serializers
from .models import Order

class OrderSerializer(serializers.ModelSerializer):
    special_instructions = serializers.CharField(required=False, allow_blank=True)
    coupon_code = serializers.CharField(required=False, allow_blank=True)
    
    class Meta:
        model = Order
        fields = ['id', 'customer', 'items', 'payment_method', 'special_instructions', 'coupon_code']
    
    def validate(self, data):
        """
        Apply conditional validation rules.
        """
        payment_method = data.get('payment_method')
        
        # If payment method is 'credit_card', validate card details
        if payment_method == 'credit_card':
            card_number = self.initial_data.get('card_number')
            card_expiry = self.initial_data.get('card_expiry')
            card_cvv = self.initial_data.get('card_cvv')
            
            if not card_number:
                raise serializers.ValidationError({
                    "card_number": "Card number is required for credit card payments."
                })
            
            if not card_expiry:
                raise serializers.ValidationError({
                    "card_expiry": "Card expiry date is required for credit card payments."
                })
            
            if not card_cvv:
                raise serializers.ValidationError({
                    "card_cvv": "Card CVV is required for credit card payments."
                })
        
        # Validate coupon code if provided
        coupon_code = data.get('coupon_code')
        if coupon_code:
            # Check if coupon exists and is valid
            if not self.validate_coupon(coupon_code):
                raise serializers.ValidationError({
                    "coupon_code": "Invalid or expired coupon code."
                })
        
        return data
    
    def validate_coupon(self, code):
        """
        Helper method to validate coupon codes.
        """
        # In a real application, you would check the coupon against a database
        valid_coupons = ['SUMMER2025', 'WELCOME10', 'FREESHIP']
        return code in valid_coupons
```

### 7. Custom Validators for Reuse

For validation logic you need to reuse across multiple serializers:

```python
# validators.py
from rest_framework import serializers
import re

def validate_phone_number(value):
    """
    Validate that the value is a valid phone number.
    """
    pattern = r'^\+?1?\d{9,15}$'
    if not re.match(pattern, value):
        raise serializers.ValidationError(
            "Phone number must be entered in the format: '+999999999'. "
            "Up to 15 digits allowed."
        )
    return value

def validate_postal_code(country, value):
    """
    Validate postal code based on country.
    """
    if country == 'US':
        pattern = r'^\d{5}(-\d{4})?$'
        if not re.match(pattern, value):
            raise serializers.ValidationError("Invalid US ZIP code.")
    elif country == 'CA':
        pattern = r'^[A-Za-z]\d[A-Za-z] \d[A-Za-z]\d$'
        if not re.match(pattern, value):
            raise serializers.ValidationError("Invalid Canadian postal code.")
    # Add more countries as needed
    return value

# serializers.py
from rest_framework import serializers
from .models import Customer, Address
from .validators import validate_phone_number, validate_postal_code

class CustomerSerializer(serializers.ModelSerializer):
    phone_number = serializers.CharField(validators=[validate_phone_number])
    
    class Meta:
        model = Customer
        fields = ['id', 'name', 'email', 'phone_number']

class AddressSerializer(serializers.ModelSerializer):
    class Meta:
        model = Address
        fields = ['id', 'customer', 'street', 'city', 'state', 'country', 'postal_code']
    
    def validate(self, data):
        country = data.get('country')
        postal_code = data.get('postal_code')
        
        if country and postal_code:
            validate_postal_code(country, postal_code)
        
        return data
```

### 8. Validation with Context

Use the serializer context to access request data or other contextual information:

```python
# views.py
from rest_framework import viewsets
from .models import Article
from .serializers import ArticleSerializer

class ArticleViewSet(viewsets.ModelViewSet):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    
    def get_serializer_context(self):
        context = super().get_serializer_context()
        context['user'] = self.request.user
        return context

# serializers.py
from rest_framework import serializers
from .models import Article

class ArticleSerializer(serializers.ModelSerializer):
    class Meta:
        model = Article
        fields = ['id', 'title', 'content', 'status', 'author']
    
    def validate_status(self, value):
        """
        Only editors and admins can publish articles.
        """
        if value == 'published':
            user = self.context.get('user')
            
            if not user or not user.is_authenticated:
                raise serializers.ValidationError(
                    "Authentication required to publish articles."
                )
            
            if not (user.is_staff or user.groups.filter(name='Editors').exists()):
                raise serializers.ValidationError(
                    "You don't have permission to publish articles."
                )
        
        return value
    
    def validate(self, data):
        """
        Set the author to the current user for new articles.
        """
        if not self.instance:  # Only for create, not update
            user = self.context.get('user')
            
            if user and user.is_authenticated:
                data['author'] = user
            else:
                raise serializers.ValidationError({
                    "author": "Authentication required to create articles."
                })
        
        return data
```

## Notes

1. **Validation Order**:
   - Validation is performed in this order:
     1. Field-level validation (e.g., `validate_title`)
     2. Validators specified in the field definition
     3. Object-level validation (`validate` method)

2. **Error Response Format**:
   - For field-specific errors, use:
     ```python
     raise serializers.ValidationError({"field_name": "Error message"})
     ```
   - For non-field errors, use:
     ```python
     raise serializers.ValidationError({"non_field_errors": ["Error message"]})
     ```
   - For simple string errors (these will be added to `non_field_errors`):
     ```python
     raise serializers.ValidationError("Error message")
     ```

3. **Built-in Validators**:
   - `UniqueValidator`: Ensures a field value is unique
   - `UniqueTogetherValidator`: Ensures a combination of fields is unique
   - `MaxValueValidator`/`MinValueValidator`: Validate numeric ranges
   - `MaxLengthValidator`/`MinLengthValidator`: Validate string lengths
   - `RegexValidator`: Validate strings against a regex pattern
   - `EmailValidator`: Validate email addresses
   - `URLValidator`: Validate URLs

4. **Performance Considerations**:
   - For database-related validations, try to perform a single query
   - Be cautious of validation logic that might be expensive
   - Consider deferring complex validation to a task queue for very large datasets

5. **Security**:
   - Validate inputs thoroughly to prevent security issues
   - Consider using DRF's built-in validators for common security concerns
   - Remember that validation is a key part of your security strategy

6. **Handling Partial Updates**:
   - Be careful with object-level validation during partial updates
   - Use `self.partial` to check if the update is partial
   - Consider required fields when validating partial updates:
     ```python
     def validate(self, data):
         if not self.partial:
             # Only apply this validation for full updates
             if 'required_field' not in data:
                 raise serializers.ValidationError({"required_field": "This field is required"})
         return data
     ```

7. **Testing Validation**:
   - Write test cases for all validation rules
   - Test both valid and invalid inputs
   - Test edge cases for numeric and date validation
