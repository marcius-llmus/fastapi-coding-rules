# FastAPI Application Architecture Rules

I created this architecture for our LLM rules. It uses the repository pattern with a service layer for business logic.
I tried to enforce a rule where service layers have a clear interface that callers must respect. For example, I'm using Pydantic models as DTOs to define the inputs.
This does make things more coupled. Often, if a service wants to call another, it may have to import schemas from the other app to meet the input requirements. (This won't happen all the time, as most cross-service calls will probably use primitive types like int or str, so we won't have to build objects manually).
The risk of circular imports also exists, especially with nested cross-service models, so it's important to maintain one-way nesting. But in general, I think this approach is good enough for our needs.
---

### Project Structure

- **Feature-Based Applications**: Organize code into applications, each in its own top-level directory within `app/` (e.g., `app/users/`, `app/wallets/`).

- **Application Contents**: Each application directory must be a self-contained module and include the following files:
  - `models.py`: SQLAlchemyModel or database table definitions.
  - `schemas.py`: Pydantic schemas for API input/output.
  - `repositories.py`: The Data Access Layer (DAL).
  - `services.py`: The pure business logic layer.
  - `routes.py`: The API (HTTP) layer.
  - `dependencies.py`: Service provider functions for dependency injection.
  - `exceptions.py`: Custom domain-specific exceptions.
  - `enums.py`: Domain-specific enumerations.
  - `tasks.py`: Periodic or recurring background tasks.

- **Shared Commons**: Use `app/commons/` for truly generic, cross-cutting concerns.
  - `app/commons/dependencies.py`: Contains primitive dependency providers like `get_db` or `get_cache`.
  - `app/commons/utils.py`: Contains generic utility functions.

---

### Service Layer

- **Decoupling**: The service layer contains the core business logic and is decoupled from the web layer. It MUST NOT have any knowledge of FastAPI or HTTP concepts like `Request` or `HTTPException`.

- **Pydantic Models as Contracts**: Services use Pydantic models (`...Create`, `...Update`, `...Read`) as their primary data contract for function signatures. These models are shared with the API (Route) Layer. This is a deliberate choice to leverage Pydantic's validation and type-hinting capabilities directly within the business logic.

- **Transaction Unaware**: The service layer is **transaction-unaware**. It operates within the single transaction provided by the `get_db` dependency but does not control its lifecycle (commit, rollback).
  - **DO NOT** call `db.commit()` or `db.rollback()`. This is handled by the `get_db` dependency.

- **No HTTP**: Do not import `fastapi`, `starlette`, or use `HTTPException` within a service.

- **Dependency Injection**: All service dependencies (e.g., repository, other services) must be explicitly injected via the `__init__` method.

- **Domain Exceptions**: Service methods MUST raise custom domain-specific exceptions defined in `exceptions.py`, never `HTTPException`.

- **Orchestration Role**: The service layer orchestrates business processes. For creation, it may construct repository-specific schemas from various inputs. For updates, it applies business rules to an existing ORM object using data from an incoming Pydantic model.


```python
# Example: app/wallets/services.py
from uuid import UUID
from app.users.models import User
from app.wallets.models import Wallet
from app.wallets.schemas import WalletUpdate # This is the schema for the REPOSITORY
from app.wallets.exceptions import WalletNotFoundException, WalletAuthorizationException
from app.wallets.repositories import WalletRepository

class WalletService:
    def __init__(self, wallet_repository: WalletRepository, current_user: User):
        self.wallet_repository = wallet_repository
        self.current_user = current_user

    def get_wallet_by_id(self, wallet_id: UUID) -> Wallet:
        wallet = self.wallet_repository.get(wallet_id)
        if not wallet:
            raise WalletNotFoundException("Wallet not found.")

        if wallet.owner_id != self.current_user.id:
            raise WalletAuthorizationException("User not authorized for this wallet.")

        return wallet

    # Example of a service method accepting a Pydantic model
    def update_wallet(
        self, *, wallet_id: UUID, wallet_in: WalletUpdate
    ) -> Wallet:
        """
        Updates a wallet. The service is responsible for fetching the object
        before applying the update logic.
        """
        # The service first fetches/authorizes the object.
        db_obj = self.get_wallet_by_id(wallet_id)

        return self.wallet_repository.update(db_obj=db_obj, obj_in=wallet_in)
```
---

### Repository Layer (Data Access Layer - DAL)

- **Purpose**: The repository's sole responsibility is to mediate between the service layer and the database. It encapsulates all data access logic for a model.

- **Location**: Each application should have a `repositories.py` file (e.g., `app/wallets/repositories.py`).

- **Purity**: Repositories are pure. They must not have any knowledge of the web layer (HTTP) or business logic. Their only dependency is the `db: Session`.

- **Dependency Flow**: The Service depends on the Repository. The Repository depends on the `db` session. A repository must never depend on a service.

- **Transaction Management**: The Repository Layer is **transaction-unaware**. It operates on the session provided by the dependency injection system but does not control the transaction lifecycle.
  - **DO** call `db.add(obj)`, `db.delete(obj)`, and `db.flush()` as needed. `flush()` is used to obtain generated values like IDs before the transaction is committed by `get_db`.
  - **DO NOT** call `db.commit()` or `db.rollback()`. The `get_db` dependency owns the transaction.

- **CRUD Method Implementation**:
  - **`get(pk: Any) -> ModelType | None`**: Retrieves a single object. Read-only.
  - **`create(obj_in: SchemaType) -> ModelType`**: Creates a model instance, calls `db.add(instance)`, and returns the transient instance. No commit.
  - **`update(db_obj: ModelType, obj_in: SchemaType) -> ModelType`**: Updates a model instance with new data. Returns the dirty instance. No commit.
  - **`delete(pk: Any) -> ModelType | None`**: Fetches an object, calls `db.delete(obj)`, and returns the deleted object. No commit.

- **Partial Updates (`update` method)**: The `update` method in the repository MUST accept a Pydantic update schema directly. It is the repository's responsibility to call `obj_in.model_dump(exclude_unset=True)` internally. This centralizes the partial update logic and ensures that only explicitly provided fields are updated, preventing accidental overwrites with `None` or default values.

- **Flushing and Refreshing**: In `create` and `update` methods, you MUST call `db.flush()` followed by `db.refresh(db_obj)`. This sends the current changes to the database and retrieves any new state (like auto-generated IDs or default values) without committing the transaction. This makes the new state available to the service layer within the same unit of work. In `delete`, `db.flush()` ensures foreign key constraints are checked immediately.

- **Repositry is not limited to models**: A repository can accept as many arguments as it need to execute its action. Let's say that the object needs a `current_user_id` to create the object that is not supplied by the pydantic model. Repository will request it in its argumetns so the service has to provide. In the repository it can be merged to the final data as `db_obj = Wallet(current_user_id=current_user_id, **obj_in.model_dump())`. Same thing, but adapted as needed for update or queries etc..

- **Update Workflow**: The standard pattern for an update is:
  1. The **Service** fetches the full ORM object using the repository's `get()` method.
  2. The **Service** applies business logic (e.g., authorization checks) on the fetched object.
  3. The **Service** calls the repository's `update()` method, passing the original ORM object and the Pydantic `UpdateSchema` it received from the API layer.

- **Pessimistic Locking (`SELECT FOR UPDATE`)**: For high-concurrency operations that require a "read-modify-write" pattern (e.g., checking a balance then debiting it), the repository is the correct place to implement row-level locking. Use SQLAlchemy's `with_for_update()` method to lock the selected rows until the transaction is committed or rolled back.
```python
  def get_for_update(self, pk: UUID) -> Wallet | None:
      return self.db.query(Wallet).filter(Wallet.id == pk).with_for_update().one_or_none()
```
- **Encapsulation**: A repository is a private implementation detail of its application. A service from one application (e.g., `UserService`) must NOT directly import and use a repository from another application (e.g., `WalletRepository`). All inter-application communication must happen at the service layer.

- **Abstract Base Class for Interface Consistency**: To ensure a consistent interface, all repositories MUST inherit from a common `BaseRepository` abstract class located in `app/commons/repositories.py`. This class defines the standard CRUD method signatures (`get`, `create`, `update`, `delete`) that every concrete repository must implement.

- **Concrete Schemas for Type Safety**: In each application's `schemas.py`, you MUST define explicit Pydantic schemas for creation and update operations (e.g., `WalletCreate`, `WalletUpdate`). The `create` and `update` methods in your concrete repository MUST use these specific schemas in their signatures. This provides strong type-checking and clarity, avoiding complex generics.

```python
# Example: An abstract base repository and a specific implementation

# app/commons/repositories.py
from abc import ABC, abstractmethod
from typing import Any, TypeVar
from sqlalchemy.orm import Session
from app.core.db import Base  # Assuming a shared declarative base
from pydantic import BaseModel

ModelType = TypeVar("ModelType", bound=Base)

class BaseRepository(ABC):
    def __init__(self, db: Session):
        self.db = db

    @abstractmethod
    def get(self, pk: Any) -> ModelType | None:
        """Gets a single record by its primary key."""
        raise NotImplementedError
...
```
```python
# app/wallets/schemas.py
from pydantic import BaseModel

class WalletCreate(BaseModel):
    currency: str
    balance: float

class WalletUpdate(BaseModel):
    balance: float | None = None
```
```python
# app/wallets/repositories.py
from uuid import UUID
from sqlalchemy.orm import Session
from app.commons.repositories import BaseRepository
from .models import Wallet
from .schemas import WalletCreate, WalletUpdate

class WalletRepository(BaseRepository):
    def get(self, pk: UUID) -> Wallet | None:
        """Gets a single wallet by its primary key."""
        return self.db.get(Wallet, pk)

    def create(self, obj_in: WalletCreate) -> Wallet:
        """Creates a new wallet, flushes to get DB-generated values, but does not commit."""
        db_obj = Wallet(**obj_in.model_dump())
        self.db.add(db_obj)
        self.db.flush()
        self.db.refresh(db_obj)
        return db_obj

    def update(self, *, db_obj: Wallet, obj_in: WalletUpdate) -> Wallet:
        """Updates a wallet, flushes to get DB-generated values, but does not commit."""
        update_data = obj_in.model_dump(exclude_unset=True)
        for key, value in update_data.items():
            setattr(db_obj, key, value)
        self.db.add(db_obj)
        self.db.flush()
        self.db.refresh(db_obj)
        return db_obj

    def delete(self, *, pk: UUID) -> Wallet | None:
        """Deletes a wallet from the session, but does not commit."""
        db_obj = self.db.get(Wallet, pk)
        if db_obj:
            self.db.delete(db_obj)
            self.db.flush()
        return db_obj

    # Example of a custom query method (read-only)
    def find_by_currency(self, currency: str) -> list[Wallet]:
        return self.db.query(Wallet).filter(Wallet.currency == currency).all()
```
---

### Dependency Injection

- **Centralized Transaction Management**: The transaction lifecycle (commit, rollback, close) MUST be handled centrally within the `get_db` dependency provider. This treats the entire HTTP request as a single, atomic unit of work, ensuring data consistency.

- **Transaction Isolation**: Each incoming request is handled in its own isolated database transaction. Operations like `db.flush()` within one request are not visible to other concurrent requests until the transaction is committed. This prevents race conditions, such as two users being assigned the same database-generated ID.

- **Chained Providers**: The dependency injection system should be structured as a chain. Each major component (Repository, Service) within an application must have its own dedicated provider function in `dependencies.py`.

- **Provider Responsibility**: A provider's only job is to instantiate its corresponding class. It does this by `Depends`-ing on the provider functions for its own dependencies. For example, `get_wallet_service` will depend on `get_wallet_repository`.

- **Granular Overrides**: This chained approach allows for highly granular mocking during testing. You can override a low-level provider (like a repository) while leaving the rest of the dependency chain intact.

- **Location of Providers**:
  - `get_db` must be defined in `app/commons/dependencies.py`.
  - Application-specific providers (e.g., `get_wallet_repository`, `get_wallet_service`) must be in the application's `dependencies.py`.
  - Request Guards like `get_current_user` belong to the application they are tied to (e.g., `app/users/dependencies.py`).

```python
# Example: `app/commons/dependencies.py`
from sqlalchemy.orm import Session
from app.core.db import engine

def get_db():
    with Session(engine) as session:
        try:
            yield session
            # If the request handler completes without exceptions, commit the transaction.
            # This check makes read-only requests slightly more efficient.
            if session.dirty or session.new or session.deleted:
                session.commit()
        except Exception:
            # If any exception occurs, roll back the entire transaction.
            session.rollback()
            raise
        # The 'with' statement ensures the session is always closed.
```

```python
# Example: app/wallets/dependencies.py
from fastapi import Depends
...
from app.users.dependencies import get_current_user  # Import from the specific app
...
def get_wallet_repository(db: Session = Depends(get_db)) -> WalletRepository:
    """Provider for the WalletRepository."""
    return WalletRepository(db=db)

def get_wallet_service(
    repository: WalletRepository = Depends(get_wallet_repository),
    current_user: User = Depends(get_current_user),
) -> WalletService:
    """Provider for the WalletService, depends on the repository provider."""
    return WalletService(wallet_repository=repository, current_user=current_user)
```
---

### Route Layer (API)

- **Translation and Unpacking Layer**: The Route Layer's primary responsibility is to translate HTTP requests into pure service layer calls. It is the boundary between the web world and the pure business logic domain.

- **Single Service Dependency**: A route should depend on a single service provider function.

- **Route Responsibilities**:
  1. Parse request data (path parameters, query, body).
  2. Use Pydantic (via FastAPI's dependency injection) to **validate** the incoming data. This can be a full schema for a `POST`/`PUT` body or individual type hints for `GET` parameters.
  3. Use `Depends` to get a service instance.
  4. **Pass the parsed identifiers and the validated Pydantic schema object** to the service layer.
  5. Catch domain exceptions from the service and translate them to `HTTPException`.
  6. Return the SQLAlchemy ORM object from the service, which FastAPI will serialize into the correct `response_model`.

- **Automatic Validation and Serialization**:
  - **Input Validation**: FastAPI automatically validates incoming request bodies against the Pydantic schema type-hinted in the route function (e.g., `wallet_in: WalletCreate`). If validation fails, it returns a `422 Unprocessable Entity` error before your code runs. Your service layer can and **must** assume it will always receive a structurally valid Pydantic schema.
  - **Output Serialization**: The service and repository layers return SQLModel ORM objects. The route layer receives this ORM object. FastAPI then uses the `response_model` defined in the route decorator (e.g., `response_model=WalletRead`) to automatically serialize the ORM object into the correct Pydantic schema for the JSON response.
```python
# Example: app/wallets/routes.py
from uuid import UUID
from fastapi import APIRouter, Depends, HTTPException, status
from app.wallets.dependencies import get_wallet_service
from app.wallets.exceptions import WalletAuthorizationException, WalletNotFoundException
from app.wallets.models import Wallet
from app.wallets.schemas import WalletRead, WalletUpdate  # API Input and Output schemas
from app.wallets.services import WalletService  # The pure service

router = APIRouter()

@router.put("/{wallet_id}", response_model=WalletRead)
def update_wallet(
    wallet_id: UUID,
    wallet_in: WalletUpdate,
    service: WalletService = Depends(get_wallet_service),
):
    try:
        # The route passes primitives and the validated schema to the service.
        updated_wallet = service.update_wallet(
            wallet_id=wallet_id,
            wallet_in=wallet_in
        )
        return updated_wallet
    # The route's responsibility is to translate domain exceptions into HTTP exceptions.
    except WalletNotFoundException as e:
        raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail=str(e))
    except WalletAuthorizationException as e:
        raise HTTPException(status_code=status.HTTP_403_FORBIDDEN, detail=str(e))
```
---

### Exception Handling

- **Application-Specific Exceptions**: Each application must have an `exceptions.py` file defining its custom domain exceptions.

- **Inheritance**: Exceptions should inherit from a common base for that application (e.g., `class WalletNotFoundException(WalletException):`).

---

### Inter-Service Communication

- **Rule**: Communication between services happens via their public methods, which accept Pydantic models as their data contract. A "consumer" service MUST import and manually construct the required Pydantic model (`...Create`, `...Update`) to call a "provider" service. This is possible because all modules exist within a single monolithic application.

- **Managing Circular Dependencies**: Direct circular imports between modules at definition time (e.g., `app1/models.py` imports `app2/models.py`, and `app2/models.py` imports `app1/models.py`) are fatal and forbidden. This risk is managed in two ways:
  1.  **Asymmetrical Nesting**: For one-way relationships, a "parent" Pydantic model (e.g., `IncidentRead`) can import and nest a "child" model (e.g., `ConferenceRead`), but the child model must not have a back-reference to the parent.
  2.  **Simplified `...ReadBasic` Models**: For reciprocal relationships (e.g., `Incident` <-> `Case`), each model defines a simplified, local `...ReadBasic` Pydantic model for the other. This breaks the import cycle, as neither `incident/models.py` nor `case/models.py` needs to import the other.

- **Use Orchestrator Services for Complex Interactions**: If a business process requires coordination between two or more services, create a new, higher-level 'Orchestrator' service. This service will depend on the lower-level services and manage their interaction. The lower-level services must remain unaware of each other. This preserves a clean, unidirectional dependency graph.

- **Unidirectional Flow**: Dependencies must flow in one direction. A higher-level or coordinating service can depend on a lower-level, more foundational service.

- **Pass Full ORM Objects for Mutations**: For operations that modify an existing object (like an update), the calling layer (e.g., the API route or another service) is responsible for fetching the full SQLAlchemy ORM object from the database first. This object, along with the Pydantic model containing the new data, is then passed to the service method.
  - **Justification**: This pattern keeps the service's `update` method clean and focused on business logic. It offloads the responsibility of fetching and handling "not found" errors to a dedicated dependency, making the core logic more testable and reusable. It also ensures the service operates on a fully materialized object within the current transaction.

- **Foreign Models are Read-Only**: A service must treat models from other domains as read-only. To modify the state of a foreign model, it must call a method on the authoritative service for that model. For example, `WalletService` cannot change `user.status`; it must call `user_service.change_status(user_id=user.id, new_status=UserStatus.ACTIVE)`.

- **Injection via Provider**: To use one service within another, inject it via the `__init__` method. The dependency provider is responsible for resolving and passing the sub-service instance.
```python
# Example: A UserService that needs WalletService.

# app/users/services.py
class UserService:
    def __init__(self, user_repository: UserRepository, wallet_service: WalletService):
        self.user_repository = user_repository
        self.wallet_service = wallet_service
    # ... methods that can call self.wallet_service ...
```
```python
# app/users/dependencies.py
from app.wallets.dependencies import get_wallet_service
from app.wallets.services import WalletService
from app.user.repositories import UserRepository
from app.user.services import UserService

def get_user_repository(db: Session = Depends(get_db)) -> UserRepository:
    """Provider for the UserRepository."""
    return UserRepository(db=db)

def get_user_service(
    repository: UserRepository = Depends(get_user_repository),
    wallet_service: WalletService = Depends(get_wallet_service),  # Resolve other service
) -> UserService:
    """Provider for UserService, depends on its own repo and other services."""
    return UserService(user_repository=repository, wallet_service=wallet_service)
```
---

### Using Services in Non-HTTP Contexts

- **Context-Specific Exception Handling**: The entry point must catch domain exceptions from the service and handle them in a way appropriate for its context (e.g., logging, retrying), not by raising `HTTPException`.

# Example: A background task reusing WalletService

def archive_wallet_task(user_id: int, wallet_id: UUID):

---

### Domain-Specific Enumerations

- **Rule**: All domain-specific enumerations (e.g., `OrderStatus`, `PaymentMethod`) MUST be defined in an `enums.py` file within their respective application directory (e.g., `app/orders/enums.py`). Cross-cutting or truly global enums may reside in `app/commons/enums.py`. Make sure to use StrEnum for strings. Can be added in pydantic models and SqlAlchemy Models

---

Some other reference of more advanced patterns: https://dev.to/markoulis/layered-architecture-dependency-injection-a-recipe-for-clean-and-testable-fastapi-code-3ioo
A good implementation of it: https://github.com/anmarkoulis/layered-architecture
