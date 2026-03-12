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
