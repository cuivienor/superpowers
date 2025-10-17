# Defense-in-Depth Validation: Python Examples

> **Note:** This file contains Python/Django-specific patterns. For core defense-in-depth principles, see the main SKILL.md.

## Language-Specific Patterns

Python with Django provides a mature validation stack similar to Rails. Here's how to apply defense-in-depth validation in Django applications.

## Complete Example: User Email Validation

**Scenario:** Ensure users always have valid, unique emails in a Django application

### Layer 1: Database Constraints

```python
# users/migrations/0001_initial.py
from django.db import migrations, models

class Migration(migrations.Migration):
    initial = True

    dependencies = []

    operations = [
        migrations.CreateModel(
            name='User',
            fields=[
                ('id', models.BigAutoField(
                    auto_created=True,
                    primary_key=True,
                    serialize=False,
                    verbose_name='ID'
                )),
                ('email', models.EmailField(
                    max_length=254,
                    unique=True,
                    db_index=True
                )),
                ('name', models.CharField(max_length=100)),
                ('created_at', models.DateTimeField(auto_now_add=True)),
            ],
        ),
        migrations.AddConstraint(
            model_name='user',
            constraint=models.CheckConstraint(
                check=~models.Q(email=''),
                name='email_not_empty'
            ),
        ),
    ]
```

**Why:** Database constraints protect against:
- Bulk SQL operations (`bulk_create`, `bulk_update`)
- Shell operations bypassing validations
- Race conditions with concurrent creates

### Layer 2: Model Validations

```python
# users/models.py
from django.db import models
from django.core.exceptions import ValidationError
from django.core.validators import EmailValidator
import re

class User(models.Model):
    email = models.EmailField(
        max_length=254,
        unique=True,
        db_index=True,
        validators=[EmailValidator()]
    )
    name = models.CharField(max_length=100)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        constraints = [
            models.CheckConstraint(
                check=~models.Q(email=''),
                name='email_not_empty'
            ),
        ]

    def clean(self):
        """Custom validation logic"""
        super().clean()

        # Normalize email
        if self.email:
            self.email = self.email.strip().lower()

        # Check for blocked domains
        if self.email:
            domain = self.email.split('@')[-1]
            if BlockedDomain.objects.filter(domain=domain).exists():
                raise ValidationError({
                    'email': f'Email domain not allowed: {domain}'
                })

    def save(self, *args, **kwargs):
        """Always run validation on save"""
        self.full_clean()
        super().save(*args, **kwargs)

    def __str__(self):
        return self.email


class BlockedDomain(models.Model):
    domain = models.CharField(max_length=255, unique=True)
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return self.domain
```

**Why:** Protects against:
- Invalid email formats before database hit
- Provides user-friendly error messages
- Normalizes data before validation
- Business logic validation (blocked domains)

### Layer 3: Form Validation

```python
# users/forms.py
from django import forms
from django.core.exceptions import ValidationError
from .models import User, BlockedDomain

class UserCreationForm(forms.ModelForm):
    class Meta:
        model = User
        fields = ['email', 'name']

    def clean_email(self):
        """Field-specific validation"""
        email = self.cleaned_data.get('email')

        if not email:
            raise ValidationError('Email is required')

        # Normalize
        email = email.strip().lower()

        # Check format
        if not '@' in email:
            raise ValidationError('Invalid email format')

        # Check blocked domains
        domain = email.split('@')[-1]
        if BlockedDomain.objects.filter(domain=domain).exists():
            raise ValidationError(f'Email domain not allowed: {domain}')

        return email

    def clean(self):
        """Cross-field validation"""
        cleaned_data = super().clean()
        email = cleaned_data.get('email')

        # Additional validation logic here
        return cleaned_data
```

**Why:** Protects against:
- Mass assignment vulnerabilities (only permitted fields)
- Invalid input from HTML forms
- Provides form-specific validation context

### Layer 4: View Validation

```python
# users/views.py
from django.http import JsonResponse
from django.views.decorators.http import require_http_methods
from django.views.decorators.csrf import csrf_exempt
from django.core.exceptions import ValidationError
from .forms import UserCreationForm
from .models import User
import json
import logging

logger = logging.getLogger(__name__)

@require_http_methods(["POST"])
@csrf_exempt  # In production, use proper CSRF protection
def create_user(request):
    """API endpoint to create user"""
    try:
        # Parse request data
        data = json.loads(request.body)

        # Layer 4: View-level validation
        if not data.get('email'):
            return JsonResponse({
                'error': 'Email is required'
            }, status=400)

        # Debug logging
        logger.info('Creating user', extra={
            'email': data.get('email'),
            'ip': request.META.get('REMOTE_ADDR'),
            'user_agent': request.META.get('HTTP_USER_AGENT'),
        })

        # Layer 3: Form validation
        form = UserCreationForm(data)
        if not form.is_valid():
            return JsonResponse({
                'error': 'Validation failed',
                'details': form.errors
            }, status=400)

        # Layer 2: Model validation (via form.save())
        user = form.save()

        return JsonResponse({
            'id': user.id,
            'email': user.email,
            'name': user.name,
        }, status=201)

    except ValidationError as e:
        return JsonResponse({
            'error': str(e)
        }, status=400)

    except json.JSONDecodeError:
        return JsonResponse({
            'error': 'Invalid JSON'
        }, status=400)

    except Exception as e:
        logger.exception('Failed to create user')
        return JsonResponse({
            'error': 'Internal server error'
        }, status=500)
```

**Why:** Protects against:
- Request-level concerns (JSON parsing, headers)
- HTTP-specific validation
- Provides proper error responses
- Audit logging

## Alternative: Django REST Framework

For API-heavy applications, Django REST Framework provides cleaner validation:

```python
# users/serializers.py
from rest_framework import serializers
from .models import User, BlockedDomain

class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ['id', 'email', 'name', 'created_at']
        read_only_fields = ['id', 'created_at']

    def validate_email(self, value):
        """Field-level validation"""
        if not value:
            raise serializers.ValidationError('Email is required')

        # Normalize
        value = value.strip().lower()

        # Check blocked domains
        domain = value.split('@')[-1]
        if BlockedDomain.objects.filter(domain=domain).exists():
            raise serializers.ValidationError(
                f'Email domain not allowed: {domain}'
            )

        return value

    def validate(self, data):
        """Object-level validation"""
        # Cross-field validation here
        return data


# users/views.py
from rest_framework import viewsets, status
from rest_framework.response import Response
from .models import User
from .serializers import UserSerializer
import logging

logger = logging.getLogger(__name__)

class UserViewSet(viewsets.ModelViewSet):
    queryset = User.objects.all()
    serializer_class = UserSerializer

    def create(self, request):
        """Create user with full validation"""
        # Debug logging
        logger.info('Creating user', extra={
            'email': request.data.get('email'),
            'ip': request.META.get('REMOTE_ADDR'),
        })

        # Serializer handles validation
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)

        # Model validation happens in save()
        user = serializer.save()

        return Response(
            serializer.data,
            status=status.HTTP_201_CREATED
        )
```

## Why All Four Layers in Django?

Each layer catches what others miss:

**Database constraints** catch:
- Direct SQL via `connection.cursor()`
- Bulk operations (`bulk_create` bypasses model-level validation like `clean()` and `save()` but still respects database constraints)
- Management commands bypassing models
- Race conditions

**Model validations** catch:
- Invalid formats early
- Business logic violations
- Provide friendly errors
- Normalize data

**Form/Serializer validation** catches:
- Mass assignment attacks
- Unintended field updates
- Request-level validation
- Field-specific context

**View validation** catches:
- HTTP-specific concerns
- Request parsing issues
- Audit logging
- Error response formatting

## Testing Each Layer

```python
# users/tests.py
from django.test import TestCase
from django.core.exceptions import ValidationError
from django.db import IntegrityError
from .models import User, BlockedDomain

class UserValidationTest(TestCase):
    def test_database_constraint_rejects_empty_email(self):
        """Layer 1: Database constraint catches empty email"""
        user = User(email='', name='Test')
        # Skip model validation to test database constraint
        with self.assertRaises(IntegrityError):
            user.save(force_insert=True)

    def test_model_validation_rejects_empty_email(self):
        """Layer 2: Model validation catches empty email"""
        user = User(email='', name='Test')
        with self.assertRaises(ValidationError) as cm:
            user.full_clean()

        self.assertIn('email', cm.exception.message_dict)

    def test_model_validation_blocks_domains(self):
        """Layer 2: Model validation checks blocked domains"""
        BlockedDomain.objects.create(domain='tempmail.com')

        user = User(email='test@tempmail.com', name='Test')
        with self.assertRaises(ValidationError) as cm:
            user.full_clean()

        self.assertIn('email', cm.exception.message_dict)

    def test_model_normalizes_email(self):
        """Layer 2: Model normalizes email on clean"""
        user = User(email='  TEST@EXAMPLE.COM  ', name='Test')
        user.full_clean()
        user.save()

        self.assertEqual(user.email, 'test@example.com')

    def test_duplicate_email_rejected(self):
        """Layer 1: Database uniqueness constraint"""
        User.objects.create(email='test@example.com', name='Test 1')

        user2 = User(email='test@example.com', name='Test 2')
        with self.assertRaises(ValidationError):
            user2.save()
```

## Common Django Validation Patterns

### Custom Validators

```python
from django.core.exceptions import ValidationError

def validate_email_domain(value):
    """Reusable validator for email domains"""
    if not value:
        return

    domain = value.split('@')[-1]
    allowed_domains = ['example.com', 'company.com']

    if domain not in allowed_domains:
        raise ValidationError(
            f'Email must be from allowed domains: {", ".join(allowed_domains)}'
        )

# Use in model
class User(models.Model):
    email = models.EmailField(
        validators=[EmailValidator(), validate_email_domain]
    )
```

### Model Constraints (Django 2.2+)

```python
class User(models.Model):
    email = models.EmailField()
    is_active = models.BooleanField(default=True)

    class Meta:
        constraints = [
            # Check constraint
            models.CheckConstraint(
                check=~models.Q(email=''),
                name='email_not_empty'
            ),
            # Unique constraint
            models.UniqueConstraint(
                fields=['email'],
                name='unique_email'
            ),
            # Conditional unique constraint
            models.UniqueConstraint(
                fields=['email'],
                condition=models.Q(is_active=True),
                name='unique_active_email'
            ),
        ]
```

### Signal-Based Validation

```python
from django.db.models.signals import pre_save
from django.dispatch import receiver

@receiver(pre_save, sender=User)
def validate_user_before_save(sender, instance, **kwargs):
    """Additional validation via signals"""
    # This runs even for bulk operations
    if not instance.email:
        raise ValidationError('Email required')

    # Log for debugging
    logger.info(f'About to save user: {instance.email}')
```

## Key Differences from Rails

- **Explicit `full_clean()`**: Must call `full_clean()` or override `save()` to run validations
- **Forms vs Models**: Forms provide HTML-specific validation layer
- **Migrations as code**: Constraints defined in migration files
- **Signal-based hooks**: Use signals instead of ActiveRecord callbacks
- **DRF serializers**: Separate validation layer for APIs
