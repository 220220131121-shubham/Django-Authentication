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
