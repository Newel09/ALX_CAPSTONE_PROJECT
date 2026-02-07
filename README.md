# Shop API — ALX Capstone Project

This project is a focused Django REST MVP: a Shop API that provides shopping lists, list items, simple budgeting, and optional price-suggestion integration.

Summary
- Goal: Allow authenticated users to create/manage shopping lists and items, track budgets per list, and optionally fetch price suggestions from public product APIs.
- MVP: CRUD for lists/items, user auth, budget estimation and a summary endpoint. External API integration is optional and added later.

Tech stack
- Django
- Django REST Framework
- SQLite for development (Postgres recommended for production)
- Token auth or JWT for API authentication

Core models (MVP)
- ShoppingList: id, owner (FK User), name, currency, budget_amount (Decimal, nullable), status (active/archived), created_at, updated_at
- Item: id, shopping_list (FK), name, quantity (Decimal), unit, category, estimated_unit_price (Decimal, nullable), is_bought (bool), note, created_at, updated_at
- Budget: integrated as fields on ShoppingList (budget_amount) and summary computed from items

Optional models
- ExternalProductCache: external_id, source, name, price, raw_payload (JSON), updated_at (for price suggestions caching)

API endpoints (summary)
- Auth (`accounts` app)
  - `POST /api/auth/register/` — register
  - `POST /api/auth/login/` — obtain token
  - `GET /api/auth/me/` — current user
- Lists (`lists` app)
  - `GET /api/lists/`
  - `POST /api/lists/`
  - `GET /api/lists/{id}/`
  - `PATCH /api/lists/{id}/`
  - `DELETE /api/lists/{id}/`
  - `POST /api/lists/{id}/archive/` (optional)
- Items (`items` app)
  - `GET /api/lists/{id}/items/`
  - `POST /api/lists/{id}/items/`
  - `PATCH /api/items/{id}/`
  - `DELETE /api/items/{id}/`
  - `POST /api/items/{id}/toggle-bought/`
- Budget summary
  - `GET /api/lists/{id}/summary/` — returns total estimated cost, budget, remaining amount, category totals
- Catalog (optional)
  - `GET /api/catalog/search/?q=...` — search external provider

Authentication & permissions
- All resources are user-scoped; require authentication.
- Only resource owners may modify their lists/items.

External API options (optional)
- DummyJSON Products — stable fake catalog for dev/testing
- Open Food Facts — real product data for groceries (by name/barcode)
Recommendation: add catalog integration after MVP; start with DummyJSON for predictable behavior.

Milestones & weekly plan
- Week 1 — Foundation & Auth: scaffold project, add DRF, implement `accounts` endpoints.
- Week 2 — Lists & Items: implement `ShoppingList` and `Item` models with CRUD and permissions.
- Week 3 — Budgeting & Summary: add budget fields, summary endpoint, filters and tests.
- Week 4 — Optional polish: external catalog, OpenAPI/Swagger docs, CI and deployment notes.

Acceptance criteria
- Authenticated users can create/read/update/delete their shopping lists and items.
- Budget summary endpoint returns correct totals and remaining budget.
- Tests cover core models and primary endpoints.

---

## Implementation Steps (Week-by-Week Breakdown)

### Week 1: Foundation & Authentication

#### Step 1.1: Scaffold the Django Project
1. Create a virtual environment and install dependencies:
```bash
python -m venv .venv
.venv/Scripts/activate
pip install django djangorestframework python-decouple
pip freeze > requirements.txt
```

2. Create the Django project and initial `accounts` app:
```bash
django-admin startproject shop_api .
python manage.py startapp accounts
```

3. Project structure should look like:
```
shop_api/
├── shop_api/           (project folder)
│   ├── __init__.py
│   ├── settings.py
│   ├── urls.py
│   ├── asgi.py
│   └── wsgi.py
├── accounts/           (authentication app)
│   ├── migrations/
│   ├── __init__.py
│   ├── admin.py
│   ├── apps.py
│   ├── models.py
│   ├── serializers.py
│   ├── urls.py
│   └── views.py
├── manage.py
└── requirements.txt
```

#### Step 1.2: Configure Django & DRF Settings
Update `shop_api/settings.py`:
- Add `'rest_framework'` to `INSTALLED_APPS`
- Add `'accounts'` to `INSTALLED_APPS`
- Add DRF configuration (e.g., `DEFAULT_AUTHENTICATION_CLASSES`, `DEFAULT_PERMISSION_CLASSES`)
- Set `REST_FRAMEWORK` dict with pagination, filtering, default page size

Example:
```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'rest_framework',
    'rest_framework.authtoken',
    'accounts',
]

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.TokenAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 20,
}
```

#### Step 1.3: Create User Registration & Login Endpoints
1. Create `accounts/serializers.py`:
   - `UserSerializer` — for user registration (username, email, password)
   - `LoginSerializer` — accepts username + password, returns token

2. Create `accounts/views.py`:
   - `RegisterView` (APIView, POST) — creates user and returns token
   - `LoginView` (APIView, POST) — authenticates user and returns token
   - `MeView` (APIView, GET) — returns current user info (requires auth)

3. Create `accounts/urls.py` and wire endpoints:
```python
path('api/auth/register/', RegisterView.as_view()),
path('api/auth/login/', LoginView.as_view()),
path('api/auth/me/', MeView.as_view()),
```

4. Add `accounts/urls.py` to main `shop_api/urls.py`:
```python
urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('accounts.urls')),
]
```

#### Step 1.4: Create and Run Migrations
```bash
python manage.py makemigrations
python manage.py migrate
```

#### Step 1.5: Test Auth Endpoints
1. Start the dev server:
```bash
python manage.py runserver
```

2. Test endpoints using curl or Postman:
   - `POST /api/auth/register/` with `{"username": "testuser", "email": "test@example.com", "password": "pass123"}`
   - `POST /api/auth/login/` with `{"username": "testuser", "password": "pass123"}` — should return token
   - `GET /api/auth/me/` with header `Authorization: Token <token>` — should return user info

**Week 1 Deliverable:** Auth endpoints work; users can register, login, and view their profile.

---

### Week 2: Shopping Lists & Items Models + CRUD

#### Step 2.1: Create `lists` and `items` Apps
```bash
python manage.py startapp lists
python manage.py startapp items
```

Add to `INSTALLED_APPS` in `settings.py`.

#### Step 2.2: Define Models (`lists/models.py` and `items/models.py`)

**lists/models.py:**
```python
from django.db import models
from django.contrib.auth.models import User

class ShoppingList(models.Model):
    STATUS_CHOICES = [('active', 'Active'), ('archived', 'Archived')]
    
    owner = models.ForeignKey(User, on_delete=models.CASCADE, related_name='shopping_lists')
    name = models.CharField(max_length=255)
    currency = models.CharField(max_length=10, default='GHS')
    budget_amount = models.DecimalField(max_digits=10, decimal_places=2, null=True, blank=True)
    status = models.CharField(max_length=20, choices=STATUS_CHOICES, default='active')
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    class Meta:
        ordering = ['-created_at']
    
    def __str__(self):
        return self.name
```

**items/models.py:**
```python
from django.db import models
from lists.models import ShoppingList

class Item(models.Model):
    shopping_list = models.ForeignKey(ShoppingList, on_delete=models.CASCADE, related_name='items')
    name = models.CharField(max_length=255)
    quantity = models.DecimalField(max_digits=10, decimal_places=2)
    unit = models.CharField(max_length=50, default='pcs')  # kg, pcs, pack, etc.
    category = models.CharField(max_length=100, blank=True)
    estimated_unit_price = models.DecimalField(max_digits=10, decimal_places=2, null=True, blank=True)
    is_bought = models.BooleanField(default=False)
    note = models.TextField(blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    class Meta:
        ordering = ['-created_at']
        indexes = [
            models.Index(fields=['shopping_list', 'is_bought']),
        ]
    
    def __str__(self):
        return f"{self.name} ({self.quantity}{self.unit})"
```

#### Step 2.3: Create Migrations & Migrate
```bash
python manage.py makemigrations
python manage.py migrate
```

#### Step 2.4: Create Serializers

**lists/serializers.py:**
```python
from rest_framework import serializers
from .models import ShoppingList

class ShoppingListSerializer(serializers.ModelSerializer):
    class Meta:
        model = ShoppingList
        fields = ['id', 'owner', 'name', 'currency', 'budget_amount', 'status', 'created_at', 'updated_at']
        read_only_fields = ['id', 'owner', 'created_at', 'updated_at']
```

**items/serializers.py:**
```python
from rest_framework import serializers
from .models import Item

class ItemSerializer(serializers.ModelSerializer):
    class Meta:
        model = Item
        fields = ['id', 'shopping_list', 'name', 'quantity', 'unit', 'category', 'estimated_unit_price', 'is_bought', 'note', 'created_at', 'updated_at']
        read_only_fields = ['id', 'created_at', 'updated_at']
```

#### Step 2.5: Create ViewSets & URLs

**lists/views.py:**
```python
from rest_framework.viewsets import ModelViewSet
from rest_framework.permissions import IsAuthenticated
from .models import ShoppingList
from .serializers import ShoppingListSerializer

class ShoppingListViewSet(ModelViewSet):
    serializer_class = ShoppingListSerializer
    permission_classes = [IsAuthenticated]
    
    def get_queryset(self):
        return ShoppingList.objects.filter(owner=self.request.user)
    
    def perform_create(self, serializer):
        serializer.save(owner=self.request.user)
```

**items/views.py:**
```python
from rest_framework.viewsets import ModelViewSet
from rest_framework.permissions import IsAuthenticated
from .models import Item
from .serializers import ItemSerializer

class ItemViewSet(ModelViewSet):
    serializer_class = ItemSerializer
    permission_classes = [IsAuthenticated]
    
    def get_queryset(self):
        # Only return items from lists the user owns
        return Item.objects.filter(shopping_list__owner=self.request.user)
```

**lists/urls.py:**
```python
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from .views import ShoppingListViewSet

router = DefaultRouter()
router.register(r'lists', ShoppingListViewSet, basename='shopping_list')

urlpatterns = [
    path('api/', include(router.urls)),
]
```

**items/urls.py:**
```python
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from .views import ItemViewSet

router = DefaultRouter()
router.register(r'items', ItemViewSet, basename='item')

urlpatterns = [
    path('api/', include(router.urls)),
]
```

Update main `shop_api/urls.py` to include both:
```python
urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('accounts.urls')),
    path('', include('lists.urls')),
    path('', include('items.urls')),
]
```

#### Step 2.6: Test Lists & Items CRUD
```bash
python manage.py runserver
```

Test endpoints:
- `GET /api/lists/` — list user's shopping lists
- `POST /api/lists/` — create list
- `GET /api/items/` — list items (across all user's lists)
- `POST /api/items/` — create item

**Week 2 Deliverable:** Users can manage shopping lists and items via CRUD REST endpoints.

---

### Week 3: Budgeting & Summary Endpoint + Tests

#### Step 3.1: Add Budget Summary Endpoint
Create a custom action in `ShoppingListViewSet` to return budget summary.

**lists/views.py:**
```python
from rest_framework.response import Response
from rest_framework.decorators import action
from decimal import Decimal

class ShoppingListViewSet(ModelViewSet):
    # ... existing code ...
    
    @action(detail=True, methods=['get'])
    def summary(self, request, pk=None):
        shopping_list = self.get_object()
        items = shopping_list.items.all()
        
        total_estimated = sum(
            (item.estimated_unit_price or 0) * item.quantity 
            for item in items
        )
        total_estimated = Decimal(str(total_estimated))
        
        budget = shopping_list.budget_amount or Decimal('0')
        remaining = budget - total_estimated
        
        return Response({
            'id': shopping_list.id,
            'name': shopping_list.name,
            'budget': float(budget),
            'total_estimated': float(total_estimated),
            'remaining': float(remaining),
            'item_count': items.count(),
            'bought_count': items.filter(is_bought=True).count(),
        })
```

Test endpoint:
- `GET /api/lists/{id}/summary/` — returns budget summary

#### Step 3.2: Add Toggle-Bought Action for Items
**items/views.py:**
```python
@action(detail=True, methods=['post'])
def toggle_bought(self, request, pk=None):
    item = self.get_object()
    item.is_bought = not item.is_bought
    item.save()
    return Response(ItemSerializer(item).data)
```

Test endpoint:
- `POST /api/items/{id}/toggle-bought/` — toggles is_bought flag

#### Step 3.3: Write Unit Tests

**lists/tests.py:**
```python
from django.test import TestCase
from django.contrib.auth.models import User
from .models import ShoppingList

class ShoppingListModelTest(TestCase):
    def setUp(self):
        self.user = User.objects.create_user(username='testuser', password='pass123')
    
    def test_create_shopping_list(self):
        list = ShoppingList.objects.create(
            owner=self.user,
            name='Weekly Groceries',
            budget_amount=100.00
        )
        self.assertEqual(list.name, 'Weekly Groceries')
        self.assertEqual(list.owner, self.user)
```

**items/tests.py:**
```python
from django.test import TestCase
from django.contrib.auth.models import User
from lists.models import ShoppingList
from .models import Item

class ItemModelTest(TestCase):
    def setUp(self):
        self.user = User.objects.create_user(username='testuser', password='pass123')
        self.list = ShoppingList.objects.create(owner=self.user, name='Test List')
    
    def test_create_item(self):
        item = Item.objects.create(
            shopping_list=self.list,
            name='Milk',
            quantity=2,
            unit='L',
            estimated_unit_price=5.00
        )
        self.assertEqual(item.name, 'Milk')
        self.assertFalse(item.is_bought)
```

**lists/test_views.py:**
```python
from rest_framework.test import APITestCase
from rest_framework import status
from django.contrib.auth.models import User
from .models import ShoppingList

class ShoppingListViewTest(APITestCase):
    def setUp(self):
        self.user = User.objects.create_user(username='testuser', password='pass123')
        self.client.force_authenticate(user=self.user)
    
    def test_list_shopping_lists(self):
        ShoppingList.objects.create(owner=self.user, name='Test List')
        response = self.client.get('/api/lists/')
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(len(response.data['results']), 1)
    
    def test_create_shopping_list(self):
        response = self.client.post('/api/lists/', {'name': 'New List', 'budget_amount': 50})
        self.assertEqual(response.status_code, status.HTTP_201_CREATED)
```

#### Step 3.4: Run Tests
```bash
python manage.py test
```

**Week 3 Deliverable:** Budget summary endpoint works; tests cover models and key endpoints.

---

### Week 4: Polish, Docs & Optional Features

#### Step 4.1: Add Swagger/OpenAPI Documentation
Install `drf-spectacular`:
```bash
pip install drf-spectacular
```

Add to `INSTALLED_APPS` and settings, then create schema view (see DRF docs).

#### Step 4.2: Create Comprehensive README Examples
Add API usage examples (curl commands, JSON request/response samples).

#### Step 4.3: (Optional) Add External Price Suggestions
Create `catalog/` app and integrate DummyJSON or Open Food Facts API with caching.

#### Step 4.4: Add Docker Support (Optional)
Create `Dockerfile` and `docker-compose.yml` for easy deployment.

**Week 4 Deliverable:** Full docs, CI/CD setup, and optional integrations.

---

## Running the Project

### Quick Start
```bash
# Activate venv
.venv/Scripts/activate

# Install dependencies
pip install -r requirements.txt

# Run migrations
python manage.py migrate

# Create superuser (for admin panel)
python manage.py createsuperuser

# Run dev server
python manage.py runserver

# Run tests
python manage.py test
```

### Testing with Curl
```bash
# Register
curl -X POST http://localhost:8000/api/auth/register/ \
  -H "Content-Type: application/json" \
  -d '{"username":"user1","email":"user1@example.com","password":"pass123"}'

# Login
curl -X POST http://localhost:8000/api/auth/login/ \
  -H "Content-Type: application/json" \
  -d '{"username":"user1","password":"pass123"}'

# Create shopping list (use token from login)
curl -X POST http://localhost:8000/api/lists/ \
  -H "Content-Type: application/json" \
  -H "Authorization: Token <YOUR_TOKEN>" \
  -d '{"name":"Weekly Groceries","budget_amount":100}'
```

---

**You are now ready to build the Shop API step-by-step!**

