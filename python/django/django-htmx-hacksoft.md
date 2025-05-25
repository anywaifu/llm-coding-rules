```markdown
# Django Project Guidelines (Revised)

General Project Structure and Conventions
- Use only absolute imports for internal project modules (e.g., `from my_app.services import MyService`).

Core Component Responsibilities
- Views:
    - Handle HTTP requests/responses, view-level auth, orchestrate Service calls, manage Forms, and render responses.
    - Get data (by calling Service query methods) and trigger business operations *only* via Services.
    - Manage Django Forms (e.g., using `FormView`). If valid, pass `cleaned_data` (unpacked) to a Service.
    - The `FormView` base class catches `ValidationError` from Services (in `form_valid`) and adds to form errors before calling `form_invalid`.
    - Minimal logic; focus on HTTP interface and presentation.

- Services (`app/services/`):
    - Encapsulate *all* primary business logic as class methods. They are the definitive source for domain operations, comprehensive business rule validation, data querying, and persistence.
    - Method names must be specific to their operation.
    - Perform all complex business rule validation logic directly within service methods at the beginning of their execution. This includes input sanity, instance internal consistency (e.g., ensuring `publish_date` is set if an action makes status `PUBLISHED`), cross-model consistency, and business process/workflow rules. This logic defines the conditions for the operation to proceed. Fail early by raising `ValidationError`.
    - Solely responsible for model instantiation, mutation, and persistence.
    - For new/modified instances: after service-level validation, instantiate/modify, set attributes, then **must** call `instance.full_clean()` before `instance.save()`. This triggers model field validators (which handle single-field integrity) and `Meta.constraints`.
    - Warning: `Model.objects.create()`, `bulk_create()`, and queryset `update()` bypass `save()` and `full_clean()`. For any validation beyond DB constraints, these must be avoided in favor of the service-led instantiate -> service validate -> `full_clean()` -> `save()` pattern.
    - Use model-defined constants/enums.
    - May call other Services or Interfaces.
    - Use `@transaction.atomic`. For read-modify-write, use `select_for_update()` within atomic transactions.
    - Raise `django.core.exceptions.ValidationError` or other appropriate Django exceptions.
    - Provide methods for querying and filtering data according to business requirements (e.g., `get_active_items_for_user(user)`, `get_items_by_criteria(**filters)`). These methods encapsulate queryset logic.

- Models (`app/models.py`):
    - Define data structure, fields, relationships, field-level validation (via the `validators=[...]` argument which can take model static/instance methods or functions from `utils/validators.py`), and database constraints (`Meta.constraints`).
    - For field-specific validation logic not broadly reusable as a generic utility, define a method on the model (ideally a `@staticmethod` if it doesn't need `self`, or an instance method if it does and it's passed as a string like `validators=['my_instance_method_validator']`) and pass its reference to the field's `validators` list.
    - Define states/choices using constants or Django's `TextChoices`/`IntegerChoices`.
    - `Meta.constraints` (e.g., `UniqueConstraint`, `CheckConstraint`) provide DB-level enforcement; ultimate source of declarative data integrity.
    - The `clean()` method should be **absent**. All custom instance-level validation beyond field validators or `Meta.constraints` is handled by Services before `full_clean()` is called.
    - Define common fields (e.g., `created_at`) in an abstract `BaseModel`.
    - Restrictions: No business logic, orchestration, external API calls, or direct persistence calls (beyond ORM's own `save()`). Do not place complex querying logic here; that belongs in Services.

- Forms (`app/forms.py`):
    - Handle initial user input validation (types, required, format via field definitions) and structure.
    - Inherit from `django.forms.Form`. **No `django.forms.ModelForm`**.
    - Define fields explicitly. `forms.fields_for_model()` inherits model field validators (including model methods listed in `validators=[...]` and functions from `utils/validators.py`). This provides immediate user feedback.
    - Custom `clean()` and `clean_<fieldname>()` methods should be **absent**. All complex cross-field input validation and business rule validation is handled by the Service.
    - Restrictions: No model saving, business logic. Database queries are generally disallowed; an exception is for populating field choices (e.g., `forms.ModelChoiceField`), which should ideally still source its queryset from a Service method designed for fetching choice data. Pass `cleaned_data` to Services.

- Utility Validators (`utils/validators.py`):
    - Contain highly reusable, often domain-agnostic, standalone validation utility functions.
    - These are typically single-field validators (take one value, for `ModelField/FormField.validators`) or generic multi-input utility functions (can be called by model validation methods or Services *if the utility is for a check not covered by model field validation*).
    - Functions accept specific data as named arguments.
    - Raise `django.core.exceptions.ValidationError`.
    - Aim for pure functions: no side effects, DB queries, or external calls.

- Interfaces (`app/interfaces/`):
    - Abstract external API communication.
    - Wrap client libraries/SDKs.
    - Translate external errors if needed.
    - Called *only* by Services.
    - Restrictions: No internal business logic.

- Tasks (`app/tasks.py`):
    - Define and execute background jobs (e.g., Celery).
    - Minimal logic. Primary role: call a **single Service method**.
    - Called Service method orchestrates validation and further operations.
    - Pass simple identifiers or primitive data types.

- Templates (`templates/`):
    - Handle presentation logic (DTL).
    - Use `base.html`, app-specific templates, `{% include %}` for partials (especially for HTMX).


- Templating: Django Template Language (DTL). Use `base.html`, app-specific templates (`templates/<app_name>/`), and `{% include %}` extensively for reusable components/partials (e.g., `templates/includes/nav.html`, `app/templates/app/includes/card.html`). Organize includes logically.
- Styling: Tailwind CSS v3 utility classes directly in templates. Use a single compiled CSS file linked in `base.html`. Avoid inline `style="..."`. Configure and compile via standard Tailwind tooling.
- Interactivity (Primary): **HTMX**.
    - Design UI as composable components (partials).
    - Use `hx-*` attributes in templates to trigger requests to specific view endpoints.
    - Django Views respond to HTMX requests by calling appropriate services (for data or actions) and rendering/returning only the necessary HTML fragment (component partial) using DTL.
    - Emphasize clean, organized HTML structure and logical HTMX attributes for maintainability and seamless integration between frontend triggers and backend view responses. Aim for a cohesive user experience despite partial page updates.
- JavaScript (Secondary): Minimize custom JS. Use only when behavior is not feasible or overly complex with HTML/CSS/HTMX. Consider Alpine.js for small, isolated component state/behavior if needed, but prefer HTMX-driven solutions. Place JS in static files (e.g., `static/js/app.js`) and bundle/import appropriately.
- Assets: Use Django `staticfiles` (`{% static %}`) for managing all static assets (CSS, JS, images).

-----
Examples
-----
-------------------
`app/models/base.py`
"""python
class BaseModel(models.Model):
    created_at = models.DateTimeField(db_index=True, default=timezone.now)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        abstract = True
"""
-------------------
`utils/validators.py`
"""python
def validate_must_be_positive(value):
    if value is not None and value <= 0:
        raise ValidationError("This value must be positive.")

def validate_string_is_alphanumeric(text_value):
    if text_value and not text_value.isalnum():
        raise ValidationError("May only contain letters and numbers.")
"""
-------------------
`app/models/some_model.py`
"""python
class SomeModel(BaseModel):
    class Status(models.TextChoices):
        DRAFT = 'DRAFT', 'Draft'
        PUBLISHED = 'PUBLISHED', 'Published'

    @staticmethod
    def validate_title_no_forbidden_words(value):
        forbidden_list = ["forbiddenword", "anotherbadone"]
        if value and any(word in value.lower() for word in forbidden_list):
            raise ValidationError("Title contains a forbidden word.")

    title = models.CharField(
        max_length=100, 
        validators=[validate_title_no_forbidden_words, validate_string_is_alphanumeric]
    )
    value_score = models.IntegerField(validators=[validate_must_be_positive])
    publish_date = models.DateField(null=True, blank=True)
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    status = models.CharField(max_length=20, choices=Status.choices, default=Status.DRAFT)

    class Meta:
        constraints = [
            models.UniqueConstraint(fields=['title', 'user'], name='unique_title_per_user_somemodel')
        ]

    def __str__(self):
        return self.title
"""
-------------------
`app/forms/some_forms.py`
"""python
some_model_form_fields = forms.fields_for_model(
    SomeModel,
    fields=['title', 'value_score', 'publish_date']
)

class CreateItemForm(forms.Form):
    title = some_model_form_fields['title']
    value_score = some_model_form_fields['value_score']
    publish_date = some_model_form_fields['publish_date']
    send_notification_immediately = forms.BooleanField(required=False)
"""
-------------------
`app/services/user_permission_service.py`
"""python
class UserPermissionService:
    @staticmethod
    def can_user_publish_items(user: User) -> bool:
        return user.is_staff or user.groups.filter(name='Publishers').exists()
"""
-------------------
`app/services/item_lifecycle_service.py`
"""python
class ItemLifecycleService:
    @classmethod
    @transaction.atomic
    def create_new_item_entry(
        cls,
        user_id: int,
        title: str,
        value_score: int,
        publish_date: date | None = None,
        send_notification_immediately: bool = False,
        **kwargs
    ) -> SomeModel:
        try:
            user = User.objects.get(pk=user_id)
        except User.DoesNotExist:
            raise ObjectDoesNotExist(f"User {user_id} not found.")

        current_status_for_item = SomeModel.Status.DRAFT
        if publish_date:
            current_status_for_item = SomeModel.Status.PUBLISHED

        if not UserPermissionService.can_user_publish_items(user) and current_status_for_item == SomeModel.Status.PUBLISHED:
            raise ValidationError("You do not have permission to publish items.")

        if current_status_for_item == SomeModel.Status.PUBLISHED and not publish_date:
             raise ValidationError({'publish_date': "Publish date is required for published items."})

        if publish_date and publish_date < date.today():
            raise ValidationError({'publish_date': "Publish date cannot be in the past."})
        
        if send_notification_immediately and current_status_for_item != SomeModel.Status.PUBLISHED:
            raise ValidationError(
                "Cannot send notification immediately if item is not being published."
            )
        
        instance = SomeModel(
            user=user,
            title=title,
            value_score=value_score,
            publish_date=publish_date,
            status=current_status_for_item
        )
        
        instance.full_clean()
        instance.save()

        if send_notification_immediately and instance.status == SomeModel.Status.PUBLISHED:
            pass

        return instance

    @classmethod
    def get_published_items_for_user(cls, user: User) -> QuerySet[SomeModel]:
        return SomeModel.objects.filter(user=user, status=SomeModel.Status.PUBLISHED).order_by('-publish_date')

    @classmethod
    def get_all_user_items_by_title_contains(cls, user: User, search_term: str) -> QuerySet[SomeModel]:
        return SomeModel.objects.filter(user=user, title__icontains=search_term).order_by('title')
"""
-------------------
`common/views.py`
"""python
class FormView(generic.FormView):
    def post(self, request: HttpRequest, *args: Any, **kwargs: Any) -> HttpResponse:
        form = self.get_form()
        if form.is_valid():
            try:
                return self.form_valid(form)
            except ValidationError as error:
                form.add_error(field=None, error=error)
        return self.form_invalid(form)
"""
-------------------
`app/views/item_views.py`
"""python
class CreateItemEntryView(LoginRequiredMixin, FormView):
    template_name = "app/create_item_form.html"
    form_class = CreateItemForm
    success_url = reverse_lazy('app:item_list_view_name')

    def form_valid(self, form):
        ItemLifecycleService.create_new_item_entry(
            user_id=self.request.user.id,
            **form.cleaned_data
        )
        return redirect(self.get_success_url())

class ItemEntryListView(LoginRequiredMixin, TemplateView):
    template_name = "app/item_list.html"

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        request_user: User = self.request.user 
        context['published_items'] = ItemLifecycleService.get_published_items_for_user(user=request_user)
        return context

create_item_view = CreateItemEntryView.as_view()
item_list_view = ItemEntryListView.as_view()
"""
```
