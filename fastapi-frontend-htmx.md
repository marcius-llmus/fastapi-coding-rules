# FastAPI-HTMX Architecture Rules

### Guiding Philosophy

HTMX is a presentation-layer technology. Its integration must not compromise the integrity or purity of our core backend architecture. The Service and Repository layers MUST remain entirely unaware of HTML, templates, or the HTMX protocol. The HTMX Route Layer acts as a dedicated presentation adapter, translating service layer outputs into human-readable views.

---

### Project Structure

To maintain a strict separation between machine-readable APIs and human-readable interfaces, each application will have a dedicated `routes` directory.

- **`routes/htmx.py`**: Contains the `APIRouter` for all HTML-rendering endpoints.
- **`routes/api.py`**: Contains the `APIRouter` for all JSON-producing endpoints.
- **`app/templates/`**: A single, top-level directory for all Jinja2 templates. Inside, templates are organized by application:
  - **`pages/`**: Contains templates for full pages, which act as the main entry point and layout shell.
  - **`partials/`**: Contains templates for components or fragments that can be loaded dynamically or included in pages.
  - **`partials/components/`**: A conventional location for smaller, reusable "building block" partials (e.g., a single item in a list) that are included by other templates.

---

### Core Patterns and Rules

#### 1. Rule: Full Pages Must Render Completely on Initial Load (The Aggregator Page Pattern)

To provide the best user experience and avoid content "pop-in," a full-page request must be served with all its initial data in a single response. We will NOT use `hx-trigger="load"` for the initial rendering of page content on a fresh page load.

- **Orchestrator Service**: The Service responsible for a full page must act as an orchestrator. It will aggregate all necessary data from other services. These services are "dumb". They just centralize data
- **Template Includes**: The full-page template will use `{% include %}` to render the necessary partials, passing the aggregated data down as context.

Example: Service for a dashboard page.
```python
# app/dashboard/services.py
class DashboardService:
    def __init__(self, wallet_service: WalletService, user_service: UserService):
        self.wallet_service = wallet_service
        self.user_service = user_service

    def get_dashboard_page_data(self) -> dict:
        wallets = self.wallet_service.get_recent_wallets()
        user_profile = self.user_service.get_profile_summary()
        return {"wallets": wallets, "user": user_profile}
```

Example: The full-page template uses `include`.

```jinja
# app/templates/dashboard/pages/main.jinja2
<div id="wallets-list">
    {% include "dashboard/partials/wallets_list.jinja2" with context %}
</div>
<div id="profile-summary">
    {% include "dashboard/partials/profile_summary.jinja2" with context %}
</div>
```

#### 2. Rule: Use Separate, Explicit Endpoints for Pages and Partials

We will NOT use the dual-purpose `full_template_name` argument in the `@htmx` decorator. Every view, whether a full page or a partial fragment, will have its own dedicated endpoint. This enforces the Single Responsibility Principle at the route layer.

- **Page Endpoint**: Renders a template from the `pages/` directory. Its service aggregates all data needed for the initial view.
- **Partial Endpoint**: Renders a template from the `partials/` directory. Its service fetches only the data needed for that specific component. It is used for all subsequent dynamic updates (e.g., refreshing a list).

#### 3. Rule: `POST` Endpoints Return Only the Created Item's Fragment

When adding a new item to a list, the `POST` endpoint must NOT return the entire updated list. It must create the item and then render and return the HTML fragment for only the newly created item.

- **Frontend Swap**: The corresponding form on the frontend will use `hx-swap="beforeend"` (or `afterbegin`, etc.) to append the new fragment to the existing list container.
- **Backend Efficiency**: This minimizes data transfer and keeps the endpoint's responsibility focused.

Example: `POST` endpoint for creating a wallet.

```python
# app/wallets/routes/htmx.py
@router.post("/", response_class=HTMLResponse, name="create_wallet")
@htmx("wallets/partials/components/wallet_item.jinja2") # Renders the single item partial
def create_wallet(
    form_data: WalletCreateForm = Depends(wallet_form_dependency),
    service: WalletService = Depends(get_wallet_service),
):
    new_wallet = service.create_wallet(wallet_in=form_data)
    return {"wallet": new_wallet}
```

#### 4. Rule: Use "Building Block" Partials for Reusability (DRY)

The HTML for a single, repeatable entity (like an item in a list) must be defined in a single "building block" partial.

- **List Assembler Partial**: A partial like `wallets_list.jinja2` is responsible for rendering a list. It does so by looping over a collection and including the building block partial for each item.
- **`POST` Endpoint Partial**: The `POST` endpoint that creates a new item will render the building block partial directly. This ensures perfect consistency and reusability.

Example: The list partial includes the item partial.

```jinja
# app/templates/wallets/partials/wallets_list.jinja2
{% for wallet in wallets %}
    {% include "wallets/partials/components/wallet_item.jinja2" %}
{% endfor %}
```

#### 5. Rule: Use Pydantic Dependencies for Form Validation

To maintain our architectural standard of using Pydantic for validation and data contracts, HTML form submissions will be mapped to Pydantic models. We will NOT use scattered `Form()` field dependencies in route signatures.

- **How it works**: FastAPI can now directly map `x-www-form-urlencoded` data to a Pydantic model using `typing.Annotated`. This removes the need for a separate dependency function, allowing for cleaner and more direct validation.
- **Form Schema**: A dedicated Pydantic model will be created in `schemas.py` to represent the form fields (e.g., `WalletCreateForm`).

Example: Using a Pydantic model directly for form validation.

```python
# app/wallets/routes/htmx.py
from typing import Annotated
from fastapi import Form

@router.post("/", ...)
def create_wallet(
    form_data: Annotated[WalletCreateForm, Form()],
    ...
):
    # The route now works with a validated Pydantic object
```
