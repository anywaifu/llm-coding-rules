# Django Project Guidelines

## Core Component Responsibilities

- Views: Handle HTTP request/response, view-level auth, orchestrate service calls, manage forms (using `BaseFormView`), render responses. Minimal logic within view methods. Get data via services.
- Services: Contain *all* primary business logic as class methods. Primary location for **business rule validation** and orchestration. Responsible for instantiating models, calling `instance.full_clean()` to trigger model validation, and handling persistence. The source of truth for domain operations and persistence logic.
- Models: Define data structure, relationships, **valid states/choices (using constants/enums)**, and **database constraints (`Meta.constraints`)**.
    - `Meta.constraints` (e.g., `UniqueConstraint`, `CheckConstraint` involving simple field comparisons) provide **database-level enforcement** and represent the **ultimate source of data integrity**.
    - The `clean()` method is crucial for validating **internal consistency** based on multiple *non-relational* fields within the model instance itself, especially for logic the database cannot easily enforce. Keep `clean()` logic simple and focused on the instance's state.
    - Define common fields (like `created_at`, `updated_at`) in an abstract `BaseModel`.
- Forms: Handle **initial user input validation** and structure (types, required, format). Provides immediate user feedback but is not the authority on business rules or data integrity. **Strictly avoid saving model instances.** Can use `forms.fields_for_model` for efficiency in defining fields.
- Interfaces: Abstract external API communication.
- Tasks: Execute background jobs, calling Service class methods.
- Templates: Handle presentation (Django Template Language), including partials for HTMX.

*Validation Flow Summary:* Forms provide initial checks -> Services enforce business rules -> Services call `model.full_clean()` (triggering `model.clean()` for internal consistency checks) -> Services call `model.save()` -> Database enforces `Meta.constraints`.

## Services (`app/services/`)

- Purpose: Encapsulate *all* business logic as class methods. Primary location for **business rule validation**. Sole responsibility for model persistence. Operates using model-defined states/choices.
- Location: `app/services.py` or `app/services/` package.
- Interaction: Call other services, perform complex business rule validation, instantiate models, set fields using **model-defined constants/choices**, **call `instance.full_clean()`** before saving, handle persistence (no `Model.objects.create()`), call interfaces.
- Transactions: Use `@transaction.atomic`.
- Error Handling: Raise `ValidationError` (field-specific dict or non-field string/`__all__`). Raise `ObjectDoesNotExist` etc. as needed.

"""python
# app/services/some_service.py
from django.db import transaction
from django.core.exceptions import ValidationError, ObjectDoesNotExist
from app.models import SomeModel, User
# ... other imports

class UserService:
    @staticmethod
    def get_user_by_id(user_id: int) -> User:
        try: return User.objects.get(pk=user_id)
        except User.DoesNotExist: raise ObjectDoesNotExist(f"User {user_id} not found.")

class SomeModelService:
    @classmethod
    @transaction.atomic
    def create_instance(cls, user_id: int, data: dict) -> SomeModel:
        user = UserService.get_user_by_id(user_id) # Fetch related data

        # 1. Business Rule Validation (Service Responsibility)
        if not cls._passes_business_rule(user, data):
             raise ValidationError("User cannot perform this action.") # Non-field error
        if data.get('value', 0) < 0: # Example simple value check better done here or via constraint
             raise ValidationError({'value': 'Value must be non-negative.'}) # Field-specific error

        # Instantiate model if all service validations pass
        instance = SomeModel(user=user, **data)
        instance.status = SomeModel.Status.PENDING_APPROVAL # Use constant

        # 2. Trigger Model Validation (clean() + Constraints via full_clean)
        # This is CRITICAL before saving.
        instance.full_clean()

        # 3. Persist (Service Responsibility)
        instance.save()
        return instance

    # ... other service methods ...
    @classmethod
    def _passes_business_rule(cls, user, data) -> bool: return True # Placeholder
"""

## Models (`app/models.py`)

- Purpose: Define DB schema, relationships, choices/constants, and **database constraints (`Meta.constraints`)**. Implement `clean()` for internal consistency checks.
- **Base Model:** Use an abstract `BaseModel` for common fields.
- **Choices/Constants:** Define choices for fields like `status`.
- **`clean()` Method:** Validate internal consistency based on multiple non-relational fields or simple instance state logic. Called by `full_clean()` in services.
- **`Meta.constraints`:** Define database-level integrity rules (e.g., `UNIQUE`, simple `CHECK`).
- Restrictions: No complex business logic spanning relations, external calls, or persistence calls within model methods/properties.

"""python
# app/models/base.py
from django.db import models
from django.utils import timezone

class BaseModel(models.Model):
    created_at = models.DateTimeField(db_index=True, default=timezone.now)
    updated_at = models.DateTimeField(auto_now=True)
    class Meta: abstract = True

# app/models/some_model.py
from django.db import models
from django.core.exceptions import ValidationError
from django.db.models import Q, F
from django.contrib.auth.models import User
from .base import BaseModel

class SomeModel(BaseModel): # Inherit from BaseModel
    class Status(models.TextChoices):
        DRAFT = 'DRAFT', 'Draft'
        PENDING_APPROVAL = 'PENDING', 'Pending Approval'
        ACTIVE = 'ACTIVE', 'Active'
        ARCHIVED = 'ARCHIVED', 'Archived'

    name = models.CharField(max_length=100)
    status = models.CharField(max_length=20, choices=Status.choices, default=Status.DRAFT)
    start_date = models.DateField(null=True, blank=True)
    end_date = models.DateField(null=True, blank=True)
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    # ... other fields ...

    class Meta:
        constraints = [
            models.UniqueConstraint(fields=['name', 'user'], name='unique_name_per_user')
        ]

    def clean(self):
        # Minimal internal consistency check (non-relational)
        if self.status == 'inactive' and self.name == 'active_special_item':
             raise ValidationError("Special item cannot have inactive status.")
        super().clean()

    @property
    def display_name(self):
        return f"{self.name} ({self.status})"

    def __str__(self): return self.name
"""

## Views (`app/views.py`)

- Purpose: Handle HTTP requests, view auth, call services, manage forms, render responses.
- View Definition: Define view instances in `views.py` (e.g., `my_view = MyView.as_view()`).
- **Form Handling (Standard Views):**
    - **Inherit from `BaseFormView`**.
    - `BaseFormView.post` handles `form.is_valid()`, calls `form_valid` in `try...except`, adds errors from `ValidationError` back to the form, and calls `form_invalid`.
    - Implement `form_valid` focusing *only* on the success case.
- HTMX Views: Use `TemplateView`, call services in `get_context_data`, render partials.

"""python
# common/views.py (or similar location for base classes)
from typing import Any
from django.core.exceptions import ValidationError, ObjectDoesNotExist
from django.http import HttpRequest, HttpResponse
from django.views import generic
import logging

logger = logging.getLogger(__name__)

class BaseFormView(generic.FormView):
    """ Base FormView to automatically handle ValidationErrors from form_valid. """
    def post(self, request: HttpRequest, *args: Any, **kwargs: Any) -> HttpResponse:
        form = self.get_form()
        if form.is_valid():
            try:
                return self.form_valid(form)
            except ValidationError as e:
                if hasattr(e, 'message_dict'):
                    for field, messages in e.message_dict.items():
                        form.add_error(field if field != '__all__' else None, messages)
                elif hasattr(e, 'message'): form.add_error(None, e.message)
                else: form.add_error(None, str(e))
            except ObjectDoesNotExist:
                 form.add_error(None, "Related data not found. Please check your input.")
            except Exception as e:
                 logger.exception(f"Unexpected error in form processing: {e}")
                 form.add_error(None, "An unexpected error occurred. Please try again later.")
        return self.form_invalid(form)


# app/views/some_views.py
from django.views.generic import TemplateView
from django.contrib.auth.mixins import LoginRequiredMixin
from django.shortcuts import redirect, render
from django.urls import reverse_lazy
from common.views import BaseFormView # Import base view
# ... other imports
from app.forms import SomeDataForm
from app.services import SomeModelService, UserService

class CreateSomethingView(LoginRequiredMixin, BaseFormView):
    template_name = "app/create_something_form.html"
    form_class = SomeDataForm
    success_url = reverse_lazy('app:list_view')

    def form_valid(self, form):
        # Service call handles business validation, calls full_clean, and saves
        result = SomeModelService.create_instance( # Use updated service method name
            user_id=self.request.user.id, data=form.cleaned_data
        )
        return redirect(self.get_success_url())

class ItemListView(LoginRequiredMixin, TemplateView):
    template_name = "app/partials/item_list.html" # Partial template

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['items'] = SomeModelService.get_active_models_for_user(
            user_id=self.request.user.id
        )
        return context

# Define view instances for urls.py
item_list_view = ItemListView.as_view()
create_something_view = CreateSomethingView.as_view()
"""

## Forms (`app/forms.py`)

- Purpose: **Initial user input validation** and defining the structure of expected input data. **Does NOT save models.**
- Validation: Use built-in validators, `clean_<fieldname>()`, `clean()`. Raise `forms.ValidationError`.
- Restrictions: No database queries (except choices), no service calls, no complex business logic, **no model saving**.
- Use `forms.fields_for_model(ModelName, fields=[...])` to generate a dictionary of fields and assign them manually within your `forms.Form` subclass.
- **Do NOT use `forms.ModelForm`**.

"""python
# app/forms/some_forms.py
from django import forms
from app.models import SomeModel, SomeModelXYZ

# Optional: Generate fields from model for efficiency
model_input_fields = forms.fields_for_model(
    SomeModel,
    fields=['name', 'start_date', 'end_date'] # Fields relevant for input
)

model_xyz_input_fields = forms.fields_for_model(
    SomeModelXYZ,
    fields=['xyz'] # Fields relevant for input
)

class SomeDataForm(forms.Form):
    # Assign generated fields
    name = model_input_fields['name']
    start_date = model_input_fields['start_date']
    end_date = model_input_fields['end_date']
    xyz = model_xyz_input_fields['xyz']
    # Add other form-specific fields
    confirm_action = forms.BooleanField(required=True, label="Confirm?")

    # Example: Form-level cross-field check (might overlap with model clean)
    def clean(self):
        cleaned_data = super().clean()
        start = cleaned_data.get("start_date")
        end = cleaned_data.get("end_date")
        # Basic check in form for quick feedback, model clean enforces it definitively
        if start and end and end < start:
            self.add_error('end_date', "End date cannot be before start date (Form check).")
        return cleaned_data
"""

## Interfaces (`app/interfaces/`)

- Purpose: Abstract external API communication. Wrap clients/SDKs. Handle external errors.
- Restrictions: No internal business logic. Called only by Services.


## Tasks (`app/tasks.py`)

- Purpose: Define background jobs (Celery).
- Logic: Primarily call Service class methods. Pass simple IDs/primitives.
- Error Handling: Catch exceptions from services, log, handle retries (`self.retry`).

"""python
# app/tasks.py
from celery import shared_task
import logging
from app.services import SomeModelService # Import service class
from django.core.exceptions import ValidationError, ObjectDoesNotExist

logger = logging.getLogger(__name__)

@shared_task(bind=True, max_retries=3, default_retry_delay=60)
def process_item_task(self, item_id: int, user_id: int):
    try:
        logger.info(f"Task started: item {item_id} for user {user_id}")
        # Example: Call a service method
        SomeModelService.background_process(item_id=item_id, triggered_by=user_id)
        logger.info(f"Task finished: item {item_id}")
    except ValidationError as e:
        # Business logic error, usually non-retryable
        logger.error(f"Task Validation Error: item {item_id}: {e}")
    except ObjectDoesNotExist as e:
        # Data not found, maybe retry if temporary, maybe log and stop
        logger.warning(f"Task Data Not Found: item {item_id}: {e}. Stopping.")
    except Exception as e:
        # Unexpected error, retry
        logger.exception(f"Task Error: item {item_id}. Retrying...")
        try:
            self.retry(exc=e)
        except self.MaxRetriesExceededError:
            logger.error(f"Task Max Retries Exceeded: item {item_id}.")
"""

## URLs (`app/urls.py`, `project/urls.py`)

- Purpose: Map URL patterns to View instances.
- Structure: Use `include()`, `app_name`. Import pre-instantiated views.


## Atomicity

- Use `@transaction.atomic` on Service methods performing database writes or requiring consistency across multiple operations.
- Use `select_for_update()` on QuerySets *within* an atomic service method to lock specific rows and prevent race conditions during read-modify-write operations (e.g., decrementing inventory, updating status based on current value).


## Frontend Conventions

- Templating: Django Template Language (DTL). Use `base.html`, app-specific templates (`templates/<app_name>/`), and `{% include %}` extensively for reusable components/partials (e.g., `templates/includes/nav.html`, `app/templates/app/includes/card.html`). Organize includes logically.
- Styling: Tailwind CSS v3 utility classes directly in templates. Use a single compiled CSS file linked in `base.html`. Avoid inline `style="..."`. Configure and compile via standard Tailwind tooling.
- Interactivity (Primary): **HTMX**.
    - Design UI as composable components (partials).
    - Use `hx-*` attributes in templates to trigger requests to specific view endpoints.
    - Django Views respond to HTMX requests by calling appropriate services (for data or actions) and rendering/returning only the necessary HTML fragment (component partial) using DTL.
    - Emphasize clean, organized HTML structure and logical HTMX attributes for maintainability and seamless integration between frontend triggers and backend view responses. Aim for a cohesive user experience despite partial page updates.
- JavaScript (Secondary): Minimize custom JS. Use only when behavior is not feasible or overly complex with HTML/CSS/HTMX. Consider Alpine.js for small, isolated component state/behavior if needed, but prefer HTMX-driven solutions. Place JS in static files (e.g., `static/js/app.js`) and bundle/import appropriately.
- Assets: Use Django `staticfiles` (`{% static %}`) for managing all static assets (CSS, JS, images).