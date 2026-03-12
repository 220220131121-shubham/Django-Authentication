# 1. Identity and Authentication Fundamentals

## 1.1 What Authentication Actually Means

**Fact**

Authentication is the process of **verifying the identity of an entity** (usually a user or service).

Formally:

> Authentication answers the question **“Who are you?”**

Example scenario:

```
User → sends credentials → system verifies → identity confirmed
```

If verification succeeds, the system **binds the request to that identity**.

---

## 1.2 Identity vs Credential

These two are often confused but are fundamentally different.

| Concept    | Meaning                        | Example                  |
| ---------- | ------------------------------ | ------------------------ |
| Identity   | Unique identifier of an entity | username, email, user_id |
| Credential | Secret proof of identity       | password, OTP, token     |

Example login:

```
Identity: shubham@example.com
Credential: password
```

The system verifies the credential **against the stored identity record**.

---

## 1.3 Authentication Factors

Credentials fall into categories called **authentication factors**.

### 1. Knowledge factor

Something the user **knows**

Examples

```
password
PIN
security questions
```

---

### 2. Possession factor

Something the user **has**

Examples

```
phone OTP
hardware token
authenticator app
smart card
```

---

### 3. Inherence factor

Something the user **is**

Examples

```
fingerprint
face recognition
retina scan
```

---

### Multi-factor authentication (MFA)

If multiple factors are required:

```
password + OTP
```

This is called **multi-factor authentication**.

---

## 1.4 Authentication vs Authorization (Important Distinction)

These are separate subsystems.

### Authentication

Verifies **identity**

```
Login with username/password
```

### Authorization

Verifies **permissions**

```
Can this user delete posts?
```

Example flow:

```
User logs in → authenticated
User tries to delete post → authorization check
```

---

## 1.5 Authentication State

After authentication, the system must **remember the user identity** for future requests.

There are two main approaches.

### Stateful authentication

Server stores session data.

Example:

```
session_id → stored in server
```

Flow:

```
login → server creates session → browser sends session cookie
```

This is the **default Django model**.

---

### Stateless authentication

Server stores **no session state**.

The client sends identity proof each request.

Example:

```
JWT token
```

Flow:

```
login → server issues token → client sends token every request
```

---

## 1.6 Authentication System Components

Every authentication system has several core modules.

| Component               | Responsibility                      |
| ----------------------- | ----------------------------------- |
| Identity store          | database of users                   |
| Credential verification | password hashing / token validation |
| Session management      | maintain login state                |
| Access control          | permissions                         |
| Security protections    | rate limiting, hashing              |

Django implements all of these.

---

## 1.7 Minimal Authentication System Model

Conceptually:

```
User
 ├── identity (username/email)
 ├── credential (password hash)
 └── metadata
```

Verification process:

```
input_password
      ↓
hash(password + salt)
      ↓
compare with stored_hash
```

If equal → authentication success.

---

## 1.8 Failure Modes (Important)

Authentication systems must guard against:

### Brute force attacks

Repeated password guessing.

Mitigation:

```
rate limiting
account lock
captcha
```

---

### Credential database leaks

Mitigation:

```
strong hashing
salt
key stretching
```

---

### Session hijacking

Mitigation:

```
secure cookies
HTTPS
session expiration
```

---

## 1.9 Where Django Fits

Django provides a **full authentication framework implementing these concepts**.

Key subsystems:

```
User model
Password hashing
Authentication backends
Sessions
Permissions
Middleware
```

All built into:

```
django.contrib.auth
```
---

# 2. Django Authentication Architecture

Django provides a complete authentication subsystem implemented primarily in:

Django

The framework’s authentication features are located inside the package:

```
django.contrib.auth
```

This package works together with other Django subsystems to provide identity management, login state, and authorization.

---

# 2.1 Core Components of Django Authentication

Django authentication is not a single module; it is a **set of cooperating components**.

| Component              | Module                     | Responsibility          |
| ---------------------- | -------------------------- | ----------------------- |
| User model             | `auth.models`              | stores user identity    |
| Authentication backend | `auth.backends`            | verifies credentials    |
| Password hashing       | `auth.hashers`             | secure password storage |
| Permissions system     | `auth.permissions`         | authorization           |
| Session framework      | `django.contrib.sessions`  | login persistence       |
| Middleware             | `AuthenticationMiddleware` | attach user to request  |

Conceptually:

```
Request
   ↓
SessionMiddleware
   ↓
AuthenticationMiddleware
   ↓
request.user available
```

---

# 2.2 Installed Applications Required

To enable authentication, several Django apps must be present in `INSTALLED_APPS`.

Minimal configuration:

```python
INSTALLED_APPS = [
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
]
```

Each has a specific role.

| App            | Purpose                               |
| -------------- | ------------------------------------- |
| `auth`         | user accounts and permissions         |
| `contenttypes` | generic relations used by permissions |
| `sessions`     | server-side session storage           |

Without these, authentication cannot function.

---

# 2.3 Database Tables Created

Running migrations creates several tables.

Important ones:

| Table                        | Purpose                |
| ---------------------------- | ---------------------- |
| `auth_user`                  | user records           |
| `auth_group`                 | group roles            |
| `auth_permission`            | permission definitions |
| `auth_user_groups`           | group membership       |
| `auth_user_user_permissions` | direct permissions     |
| `django_session`             | active sessions        |

Simplified schema:

```
User
 ├── Groups
 └── Permissions

Session
 └── user_id
```

---

# 2.4 Middleware Integration

Authentication in Django depends on two middleware layers.

### 1. Session Middleware

```
django.contrib.sessions.middleware.SessionMiddleware
```

Responsibilities:

* read session cookie
* load session data from storage
* attach `request.session`

---

### 2. Authentication Middleware

```
django.contrib.auth.middleware.AuthenticationMiddleware
```

Responsibilities:

* reads session user id
* loads corresponding user
* attaches `request.user`

After this middleware runs:

```python
request.user
```

becomes available everywhere (views, templates, DRF, etc.).

---

# 2.5 Authentication Pipeline (Request Lifecycle)

When a request arrives at the server:

```
HTTP request
   ↓
SessionMiddleware
   ↓
AuthenticationMiddleware
   ↓
view function
```

Detailed process:

1. Browser sends cookie:

```
sessionid=abc123
```

2. Session middleware retrieves session record.

3. Session contains:

```
_auth_user_id
_auth_user_backend
```

4. Authentication middleware loads user from database.

5. User instance attached to:

```python
request.user
```

If no valid session exists:

```
request.user = AnonymousUser
```

---

# 2.6 Anonymous User

If no authentication exists, Django assigns:

```
django.contrib.auth.models.AnonymousUser
```

Properties:

| Attribute          | Value |
| ------------------ | ----- |
| `is_authenticated` | False |
| `is_staff`         | False |
| `is_superuser`     | False |

Example usage:

```python
if request.user.is_authenticated:
    ...
```

Important note:

`AnonymousUser` prevents null checks across the framework.

---

# 2.7 Login State Storage

Django stores login state in the session.

Session contains:

```
{
 "_auth_user_id": "5",
 "_auth_user_backend": "django.contrib.auth.backends.ModelBackend"
}
```

Meaning:

| Field                | Purpose                         |
| -------------------- | ------------------------------- |
| `_auth_user_id`      | user primary key                |
| `_auth_user_backend` | backend used for authentication |

This allows Django to reload the user on future requests.

---

# 2.8 Authentication Backends

Credential verification is delegated to **authentication backends**.

Default backend:

```
django.contrib.auth.backends.ModelBackend
```

Responsibilities:

* verify username/password
* load user object
* check permissions

Django supports **multiple backends simultaneously**.

Example configuration:

```python
AUTHENTICATION_BACKENDS = [
    "django.contrib.auth.backends.ModelBackend",
]
```

Examples of custom backends:

* email login
* LDAP authentication
* OAuth provider
* external identity server

---

# 2.9 High-Level Architecture Diagram

Putting everything together:

```
Client
  │
  │ request
  ▼
Django Server
  │
  ├── SessionMiddleware
  │      │
  │      └── loads session
  │
  ├── AuthenticationMiddleware
  │      │
  │      └── attaches request.user
  │
  └── View
         │
         └── business logic
```

---

# 2.10 Responsibilities Split

Django separates responsibilities clearly.

| Responsibility          | Component                 |
| ----------------------- | ------------------------- |
| User storage            | User model                |
| Credential verification | authentication backend    |
| Session persistence     | session framework         |
| User context            | authentication middleware |
| Authorization           | permission system         |

This modular design is why Django authentication is **highly extensible**.

---

# Important Observations (Architectural)

**Fact**

Django authentication is primarily **session-based and stateful**.

Implication:

* works naturally for server-rendered apps
* requires adjustments for APIs (DRF, JWT)

---

# 3. Django User Model

## 3.1 Role of the User Model

**Fact**

The **User model** represents an authenticated identity inside Django.

It is the **central data structure** used by the authentication system.

Conceptually:

```
User
 ├── identity fields
 ├── credential fields
 ├── permission flags
 └── metadata
```

The user record binds together:

* identity
* authentication credentials
* authorization properties
* session ownership

---

## 3.2 Default Django User Model

The default implementation is:

```
django.contrib.auth.models.User
```

This model is automatically created when migrations run.

Database table:

```
auth_user
```

---

## 3.3 Default Fields

The built-in user model contains several fields grouped by responsibility.

### Identity Fields

These uniquely identify the user.

| Field      | Type   | Purpose                  |
| ---------- | ------ | ------------------------ |
| `username` | string | primary login identifier |
| `email`    | string | contact identifier       |

Important note:

`username` is the **default authentication identity**.

---

### Credential Field

| Field      | Purpose         |
| ---------- | --------------- |
| `password` | hashed password |

Passwords are **never stored in plaintext**.

Example stored value:

```
pbkdf2_sha256$600000$saltsalt$hashvalue
```

Structure:

```
algorithm$salt$hash
```

---

### Status Flags

These fields control **account state**.

| Field          | Meaning                  |
| -------------- | ------------------------ |
| `is_active`    | account enabled          |
| `is_staff`     | admin site access        |
| `is_superuser` | bypass permission checks |

Example permission logic:

```
if user.is_superuser:
    allow_all_actions
```

---

### Metadata Fields

| Field         | Purpose             |
| ------------- | ------------------- |
| `first_name`  | profile             |
| `last_name`   | profile             |
| `date_joined` | account creation    |
| `last_login`  | last authentication |

These are **not used for authentication logic** but are useful for application behavior.

---

## 3.4 Database Schema Representation

Simplified table structure:

```
auth_user

id (PK)
username
password
email
first_name
last_name
is_active
is_staff
is_superuser
last_login
date_joined
```

Primary key:

```
id
```

All sessions reference the user through this id.

---

## 3.5 How Django Uses the User Model

The user object is attached to every request after authentication.

Example inside a view:

```python
def dashboard(request):
    user = request.user
```

Possible values:

```
User instance
```

or

```
AnonymousUser
```

Example usage:

```python
if request.user.is_authenticated:
    print(request.user.username)
```

---

## 3.6 Password Storage Mechanism

Django uses **secure password hashing**.

The hashing framework is located in:

```
django.contrib.auth.hashers
```

Default algorithm (modern Django versions):

```
PBKDF2 + SHA256
```

Hashing process:

```
password
   ↓
salt generated
   ↓
key stretching (PBKDF2)
   ↓
stored hash
```

Important property:

```
stored_hash ≠ password
```

Password verification process:

```
hash(input_password)
        ↓
compare with stored hash
```

---

## 3.7 Creating Users

Django provides helper methods to create users safely.

### Creating a normal user

```python
from django.contrib.auth.models import User

User.objects.create_user(
    username="shubham",
    email="shubham@example.com",
    password="securepassword"
)
```

Important:

```
create_user()
```

automatically hashes the password.

---

### Creating a superuser

```
python manage.py createsuperuser
```

This sets:

```
is_staff = True
is_superuser = True
```

---

## 3.8 Password Verification

During login Django executes roughly:

```
authenticate(username, password)
```

Internally:

```
1. find user by username
2. hash provided password
3. compare with stored hash
```

If match:

```
return user
```

Else:

```
return None
```

---

## 3.9 Accessing the User Model Properly

Direct imports are discouraged in reusable apps.

Correct approach:

```python
from django.contrib.auth import get_user_model

User = get_user_model()
```

Reason:

Django allows **custom user models**.

Hardcoding the default model breaks extensibility.

---

## 3.10 User Relationships

The user model connects to several other authentication tables.

Simplified relational diagram:

```
User
 ├── Groups (many-to-many)
 └── Permissions (many-to-many)
```

Tables involved:

```
auth_group
auth_permission
auth_user_groups
auth_user_user_permissions
```

These relationships implement **authorization**.

---

# Architectural Insight

Important system property:

```
User model = identity anchor
```

Everything references it:

```
sessions
permissions
groups
authentication backends
```

If the user model changes, **many parts of the system are affected**.

---

# 3.11 Limitation of the Default User Model

The built-in model works well for basic systems but has constraints.

Examples:

```
email login instead of username
additional profile fields
external identity providers
custom authentication logic
```

Because of this, Django supports **custom user models**.

---

# Next Logical Step

After understanding the user model, the next topic should be:

```
Custom User Models
```

because it explains:

* why the default model is often insufficient
* how Django allows identity schema changes

The progression will be:

```
User model
   ↓
Custom user models
   ↓
Authentication backends
   ↓
Login/logout flow
   ↓
Sessions
```
---

# 4. Custom User Models in Django

## 4.1 Why Custom User Models Exist

**Fact**

The default model is designed for **generic applications**, not for all real-world identity systems.

Common requirements that exceed the default schema:

| Requirement                     | Example                    |
| ------------------------------- | -------------------------- |
| Email as login identity         | email instead of username  |
| Remove username entirely        | modern SaaS systems        |
| Additional identity fields      | phone number, organization |
| Different authentication scheme | external identity provider |
| Domain-specific user metadata   | roles, departments         |

Example identity schema for a SaaS system:

```
User
 ├── email
 ├── password
 ├── organization_id
 └── role
```

The built-in `User` model cannot be modified safely after migrations.

Therefore Django provides **custom user models**.

---

# 4.2 Django's Design Strategy

Django supports two approaches.

| Method                 | Use Case               |
| ---------------------- | ---------------------- |
| **Extend User model**  | minor additions        |
| **Replace User model** | major identity changes |

Implementation mechanisms:

| Base Class         | Purpose                           |
| ------------------ | --------------------------------- |
| `AbstractUser`     | extend existing structure         |
| `AbstractBaseUser` | build identity model from scratch |

---

# 4.3 Approach 1 — Extending with `AbstractUser`

This is the **recommended approach for most projects**.

`AbstractUser` provides:

* username
* password
* permissions
* groups
* authentication logic

You simply **add fields**.

Example:

```python
from django.contrib.auth.models import AbstractUser
from django.db import models

class User(AbstractUser):
    phone_number = models.CharField(max_length=20)
    organization = models.CharField(max_length=100)
```

Resulting schema:

```
User
 ├── username
 ├── password
 ├── email
 ├── phone_number
 └── organization
```

All authentication features continue to work automatically.

---

# 4.4 Registering the Custom Model

After defining the model, Django must be told to use it.

In `settings.py`:

```python
AUTH_USER_MODEL = "accounts.User"
```

Important constraint:

**This must be defined before the first migration.**

Failure mode:

```
Changing user model after migrations
→ migration conflicts
→ foreign key corruption
```

Fixing this later can require **full database reset**.

---

# 4.5 Approach 2 — Using `AbstractBaseUser`

This approach is used when **identity logic must be redesigned**.

`AbstractBaseUser` provides only:

```
password hashing
last_login
authentication hooks
```

Everything else must be implemented manually.

Example minimal model:

```python
from django.contrib.auth.models import AbstractBaseUser
from django.db import models

class User(AbstractBaseUser):
    email = models.EmailField(unique=True)

    USERNAME_FIELD = "email"
```

You must implement:

* user manager
* permissions integration
* admin configuration
* required fields

Because of this complexity, it is **rarely necessary**.

---

# 4.6 Required Properties for `AbstractBaseUser`

Django expects several attributes.

Example:

```python
USERNAME_FIELD = "email"
REQUIRED_FIELDS = []
```

Meaning:

| Property          | Purpose                          |
| ----------------- | -------------------------------- |
| `USERNAME_FIELD`  | primary login identifier         |
| `REQUIRED_FIELDS` | required when creating superuser |

Example:

```
login identity = email
```

---

# 4.7 Custom User Manager

A manager is required to correctly create users.

Example:

```python
from django.contrib.auth.models import BaseUserManager

class UserManager(BaseUserManager):

    def create_user(self, email, password=None):
        user = self.model(email=self.normalize_email(email))
        user.set_password(password)
        user.save()
        return user
```

Important behavior:

```
set_password()
```

This ensures **secure hashing**.

If developers instead do:

```
user.password = password
```

it results in **plaintext password storage**, which is a critical security failure.

---

# 4.8 Identity Field Selection

Choosing the login identifier is an **architectural decision**.

Common patterns:

| Identity Field | Typical System    |
| -------------- | ----------------- |
| username       | legacy systems    |
| email          | modern web apps   |
| phone number   | mobile-first apps |
| external ID    | SSO providers     |

Modern SaaS pattern:

```
email + password
```

No username.

Example configuration:

```python
USERNAME_FIELD = "email"
```

---

# 4.9 Accessing the Custom User Model

Code should never hardcode:

```
django.contrib.auth.models.User
```

Instead use:

```python
from django.contrib.auth import get_user_model

User = get_user_model()
```

or in models:

```python
from django.conf import settings

settings.AUTH_USER_MODEL
```

Example foreign key:

```python
author = models.ForeignKey(
    settings.AUTH_USER_MODEL,
    on_delete=models.CASCADE
)
```

This prevents dependency breakage.

---

# 4.10 User Model and Permissions

The custom user model still integrates with Django's permission system.

Relationships remain:

```
User
 ├── Groups
 └── Permissions
```

These are provided by:

```
PermissionsMixin
```

If using `AbstractUser`, this is already included.

If using `AbstractBaseUser`, it must be added manually.

---

# 4.11 Migration Implications

Changing the user model impacts many tables.

Dependencies include:

```
sessions
admin
permissions
foreign keys
```

Therefore Django strongly recommends:

```
Define custom user model at project start
```

Even if initially identical to the default model.

---

# 4.12 Recommended Production Pattern

Most professional Django systems use:

```
AbstractUser
+
email as login identifier
```

Example:

```python
class User(AbstractUser):
    username = None
    email = models.EmailField(unique=True)

    USERNAME_FIELD = "email"
    REQUIRED_FIELDS = []
```

This produces:

```
login = email
```

while retaining Django’s authentication infrastructure.

---

# Architectural Summary

Identity layer:

```
User model
   ↓
Authentication backend
   ↓
Session framework
   ↓
Permissions system
```

The **user model defines the identity schema** used across the entire stack.

---

# 5. Django Authentication Backends and Login Flow

## 5.1 Purpose of Authentication Backends

**Fact**

Authentication backends are responsible for **verifying credentials and returning a user object**.

In Django:

```python
authenticate(...)
```

does **not directly verify credentials**.

Instead it **delegates verification to configured authentication backends**.

Conceptual model:

```
credentials
     ↓
authenticate()
     ↓
authentication backend
     ↓
user object OR None
```

This design allows Django to support **multiple authentication mechanisms simultaneously**.

Examples:

```
username + password
email + password
LDAP authentication
OAuth provider
SSO systems
```

---

# 5.2 Default Authentication Backend

Django ships with a default backend:

```
django.contrib.auth.backends.ModelBackend
```

Responsibilities:

| Responsibility          | Description                |
| ----------------------- | -------------------------- |
| credential verification | validate username/password |
| user retrieval          | load user from database    |
| permission checks       | integrate with permissions |

Configuration:

```python
AUTHENTICATION_BACKENDS = [
    "django.contrib.auth.backends.ModelBackend"
]
```

This backend authenticates using the **User model stored in the database**.

---

# 5.3 The `authenticate()` Function

Authentication always starts with:

```python
from django.contrib.auth import authenticate
```

Example usage:

```python
user = authenticate(
    request,
    username="shubham",
    password="mypassword"
)
```

Return values:

| Result        | Meaning                |
| ------------- | ---------------------- |
| `User object` | authentication success |
| `None`        | authentication failure |

Important property:

```python
authenticate()
```

**does not log the user in**.

It only verifies credentials.

---

# 5.4 Internal Backend Resolution

When `authenticate()` runs, Django executes roughly:

```
for backend in AUTHENTICATION_BACKENDS:
      user = backend.authenticate(...)
      if user:
            return user
return None
```

This means:

* multiple authentication systems can coexist
* Django stops at the **first successful backend**

Example architecture:

```
OAuthBackend
LDAPBackend
ModelBackend
```

If OAuth succeeds, the others are never checked.

---

# 5.5 `login()` Function

Authentication verification is separate from **login state creation**.

To persist login state:

```python
from django.contrib.auth import login
```

Example flow:

```python
user = authenticate(request, username, password)

if user:
    login(request, user)
```

Responsibilities of `login()`:

| Operation      | Description           |
| -------------- | --------------------- |
| create session | persistent login      |
| store user id  | inside session        |
| store backend  | authentication source |

---

# 5.6 What `login()` Stores in the Session

After login, Django writes values into the session.

Example session data:

```
{
 "_auth_user_id": "5",
 "_auth_user_backend": "django.contrib.auth.backends.ModelBackend",
 "_auth_user_hash": "session integrity hash"
}
```

Meaning:

| Key                  | Purpose            |
| -------------------- | ------------------ |
| `_auth_user_id`      | user primary key   |
| `_auth_user_backend` | backend used       |
| `_auth_user_hash`    | session validation |

These values allow Django to **restore authentication state on future requests**.

---

# 5.7 Request Lifecycle After Login

Once login succeeds, every request goes through middleware.

Lifecycle:

```
HTTP request
      ↓
SessionMiddleware
      ↓
AuthenticationMiddleware
      ↓
request.user attached
```

Process detail:

1. Browser sends cookie

```
sessionid=abc123
```

2. Session middleware loads session record.

3. Authentication middleware reads:

```
_auth_user_id
```

4. Django loads the user from the database.

5. User instance becomes available as:

```python
request.user
```

---

# 5.8 `request.user`

After middleware runs:

```python
request.user
```

contains either:

```
User instance
```

or

```
AnonymousUser
```

Example usage:

```python
if request.user.is_authenticated:
    print(request.user.username)
```

This allows views to **determine authentication state safely**.

---

# 5.9 Logout Process

Logging out removes the session authentication data.

Example:

```python
from django.contrib.auth import logout

logout(request)
```

Operations performed:

```
delete session authentication keys
invalidate session
```

After logout:

```
request.user → AnonymousUser
```

---

# 5.10 Example Login View

Minimal login implementation:

```python
from django.contrib.auth import authenticate, login
from django.shortcuts import render

def login_view(request):

    username = request.POST["username"]
    password = request.POST["password"]

    user = authenticate(request, username=username, password=password)

    if user:
        login(request, user)
```

Authentication pipeline:

```
credentials
     ↓
authenticate()
     ↓
backend verification
     ↓
login()
     ↓
session created
```

---

# 5.11 Custom Authentication Backends

Developers can implement custom authentication logic.

Example cases:

```
email login
phone number login
LDAP authentication
OAuth providers
external identity servers
```

Basic structure:

```python
class CustomBackend:

    def authenticate(self, request, username=None, password=None):
        ...
```

Backend must return:

```
User instance
```

or

```
None
```

---

# 5.12 Architectural Flow of Django Authentication

Complete pipeline:

```
User submits credentials
        ↓
authenticate()
        ↓
authentication backend
        ↓
user object returned
        ↓
login()
        ↓
session created
        ↓
middleware loads user on each request
```

This separation of responsibilities allows Django to support **multiple authentication strategies without changing application code**.

---

# Architectural Insight

Authentication responsibilities are split across layers.

| Layer                  | Responsibility          |
| ---------------------- | ----------------------- |
| User model             | identity storage        |
| hashers                | password security       |
| authentication backend | credential verification |
| login()                | session creation        |
| middleware             | attach user to request  |

This layered architecture is why Django authentication is **extensible and modular**.

---

This reflects how Django internally processes authentication.

---

# 1. Initial State Before Login

User already exists in database.

Example row in `accounts_user`:

```
id = 5
username = shubham
password = pbkdf2_sha256$600000$saltsalt$hashvalue
```

User opens login page and submits:

```
POST /login
username=shubham
password=abc123
```

Request enters Django.

---

# 2. Request Enters Django

Execution pipeline:

```
WSGI server
    ↓
Django request handler
    ↓
Middleware stack
    ↓
URL resolver
    ↓
login_view
```

Now the login view executes.

Example:

```python
user = authenticate(request, username=username, password=password)
```

This is where the **authentication call chain starts**.

---

# 3. authenticate() Entry Point

Function location:

```
django.contrib.auth.__init__.py
```

Definition:

```python
authenticate(request=None, **credentials)
```

Simplified implementation:

```python
def authenticate(request=None, **credentials):

    for backend in get_backends():

        user = backend.authenticate(request, **credentials)

        if user is not None:
            user.backend = backend_path
            return user

    return None
```

Important behavior:

```
authenticate() does NOT verify passwords itself.
```

It **delegates to authentication backends**.

---

# 4. Backend Discovery

Function:

```
get_backends()
```

This loads backends from settings:

```python
AUTHENTICATION_BACKENDS = [
    "django.contrib.auth.backends.ModelBackend"
]
```

So Django loads:

```
ModelBackend
```

---

# 5. ModelBackend.authenticate()

Location:

```
django.contrib.auth.backends.ModelBackend
```

Method:

```python
def authenticate(self, request, username=None, password=None):
```

Simplified code:

```python
UserModel = get_user_model()

try:
    user = UserModel.objects.get(username=username)
except UserModel.DoesNotExist:
    return None

if user.check_password(password):
    return user

return None
```

Execution flow:

```
lookup user by username
       ↓
verify password hash
       ↓
return user or None
```

---

# 6. Password Verification

Function called:

```
user.check_password(password)
```

Location:

```
django.contrib.auth.base_user.AbstractBaseUser
```

Implementation:

```python
def check_password(self, raw_password):

    return check_password(raw_password, self.password)
```

Now Django calls the global password utility.

---

# 7. Password Hash Verification

Function:

```
django.contrib.auth.hashers.check_password()
```

Simplified implementation:

```python
def check_password(password, encoded):

    hasher = identify_hasher(encoded)

    return hasher.verify(password, encoded)
```

Process:

```
stored hash parsed
        ↓
algorithm identified
        ↓
password hashed again
        ↓
hash comparison
```

Example stored value:

```
pbkdf2_sha256$600000$salt$hash
```

Steps executed:

```
extract algorithm
extract salt
recompute PBKDF2(password + salt)
compare hashes
```

If equal:

```
True
```

User authenticated.

---

# 8. authenticate() Returns User

Back to:

```python
authenticate()
```

Now Django attaches backend info:

```
user.backend = "django.contrib.auth.backends.ModelBackend"
```

Return value:

```
<User: shubham>
```

Important:

```
User is authenticated but NOT logged in yet.
```

---

# 9. login() Function Call

View calls:

```python
login(request, user)
```

Location:

```
django.contrib.auth.__init__.py
```

Simplified implementation:

```python
def login(request, user):

    session_auth_hash = user.get_session_auth_hash()

    request.session["_auth_user_id"] = user.pk
    request.session["_auth_user_backend"] = user.backend
    request.session["_auth_user_hash"] = session_auth_hash
```

Main responsibility:

```
store authentication state inside session
```

---

# 10. Session Creation

If session doesn't exist:

```
SessionMiddleware creates one
```

Session engine default:

```
django.contrib.sessions.backends.db
```

Session store object:

```
SessionStore
```

Example session content:

```
{
 "_auth_user_id": "5",
 "_auth_user_backend": "django.contrib.auth.backends.ModelBackend",
 "_auth_user_hash": "securehash"
}
```

---

# 11. Session Saved

Session store writes to database.

Table:

```
django_session
```

Example record:

```
session_key = 8a3c1f29ab23d
session_data = encoded dict
expire_date = 2026-03-20
```

---

# 12. Session Cookie Sent to Browser

Response header:

```
Set-Cookie: sessionid=8a3c1f29ab23d
```

Browser stores:

```
sessionid cookie
```

Login request ends.

---

# 13. Next Request (User Visits Dashboard)

Browser sends:

```
GET /dashboard
Cookie: sessionid=8a3c1f29ab23d
```

Now middleware becomes important.

---

# 14. SessionMiddleware Execution

Location:

```
django.contrib.sessions.middleware.SessionMiddleware
```

Simplified process:

```python
session_key = request.COOKIES["sessionid"]

session_data = SessionStore.load(session_key)

request.session = session_data
```

Now request contains:

```
request.session
```

Example:

```
{
 "_auth_user_id": "5",
 "_auth_user_backend": "...ModelBackend"
}
```

---

# 15. AuthenticationMiddleware Execution

Location:

```
django.contrib.auth.middleware.AuthenticationMiddleware
```

Simplified logic:

```python
user_id = request.session["_auth_user_id"]

user = User.objects.get(pk=user_id)

request.user = user
```

If session missing:

```
request.user = AnonymousUser()
```

---

# 16. View Finally Executes

Example dashboard:

```python
def dashboard(request):

    print(request.user)
```

Result:

```
<User: shubham>
```

User is authenticated.

---

# Full Django Login Call Chain

Complete runtime sequence:

```
login_view
    ↓
authenticate()
    ↓
get_backends()
    ↓
ModelBackend.authenticate()
    ↓
User.objects.get()
    ↓
check_password()
    ↓
hashers.check_password()
    ↓
authenticate() returns user
    ↓
login()
    ↓
request.session updated
    ↓
SessionStore.save()
    ↓
session cookie sent to browser
```

Next request:

```
HTTP request
    ↓
SessionMiddleware
    ↓
load session
    ↓
AuthenticationMiddleware
    ↓
load user
    ↓
request.user available
```

---

# Key Architectural Insight

Django authentication consists of **four major layers**.

| Layer                  | Responsibility          |
| ---------------------- | ----------------------- |
| User model             | identity storage        |
| Hashers                | credential security     |
| Authentication backend | credential verification |
| Session framework      | login persistence       |

These layers interact during login but remain **independently replaceable**, which is why Django authentication is highly extensible.
