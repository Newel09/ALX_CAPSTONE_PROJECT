Shop API — Feature Summary

When built, this application will provide the following capabilities:

- User authentication: register, login, and view own profile (token-based).
- Manage shopping lists: create, rename, archive, delete, and list user-owned shopping lists.
- Manage list items: add items to lists, update quantity/unit/category, mark bought/unbought, and delete items.
- Budget tracking: set an optional budget per shopping list and store estimated unit prices per item.
- Summary endpoint: return list totals, estimated cost, budget, remaining amount, and basic counts.
- Optional integrations (post-MVP): fetch price suggestions from external product APIs, share lists with roles, and basic analytics (most-bought items, category totals).

This file lists only the app capabilities; implementation details and step-by-step plan are in `PLAN.md`.
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

