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

(omitted further content for brevity)
