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
