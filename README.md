Covered scope:

```
Authentication vs Authorization
Django Auth Architecture
User Model
Password Hashing
Sessions
Login System
Logout System
Login Protection
Permissions
Groups & Roles
Built-in Auth Views
Password Reset
```

---

# 1. Authentication vs Authorization

### Authentication

User ki **identity verify** karta hai.

Question:

```
Who are you?
```

Example:

```
username + password login
```

Result:

```
User authenticated
```

---

### Authorization

User **kya kar sakta hai** decide karta hai.

Question:

```
What can you do?
```

Example:

```
Student → view course
Instructor → create course
Admin → manage users
```

---

### Flow

```
Request
   ↓
Authentication
   ↓
Authorization
   ↓
Access granted / denied
```

---

# 2. Django Authentication Architecture

Main components:

```
User Model
Authentication Backend
Sessions
Middleware
Permissions
Groups
```

Execution flow:

```
User login request
       ↓
authenticate()
       ↓
Authentication Backend
       ↓
User object returned
       ↓
login()
       ↓
Session created
       ↓
request.user available
```

Middleware responsible:

```
SessionMiddleware
AuthenticationMiddleware
```

---

# 3. Django User Model

Default model:

```
django.contrib.auth.models.User
```

Common fields:

```
username
password
email
first_name
last_name
is_staff
is_superuser
is_active
date_joined
last_login
```

Password stored as:

```
hashed value
```

Never plain text.

---

# 4. Custom User Model

Recommended practice:

```
Always create custom user model
before first migration
```

Example:

```python
from django.contrib.auth.models import AbstractUser

class User(AbstractUser):
    phone = models.CharField(max_length=15)
    role = models.CharField(max_length=20)
```

Settings:

```python
AUTH_USER_MODEL = "accounts.User"
```

---

# 5. Password Hashing

Django never stores plain passwords.

Default algorithm:

```
PBKDF2 + SHA256
```

Password format:

```
algorithm$iterations$salt$hash
```

Example:

```
pbkdf2_sha256$390000$...
```

Password set method:

```python
user.set_password(password)
```

Password check:

```python
check_password()
```

---

# 6. Sessions

Sessions store **logged-in user identity**.

Purpose:

```
User login once
Use identity across requests
```

Session storage table:

```
django_session
```

Session data example:

```
_auth_user_id
_auth_user_backend
```

Browser cookie:

```
sessionid=abc123
```

---

# 7. Login System

Login process:

```
User submits credentials
      ↓
authenticate()
      ↓
password hash verified
      ↓
User object returned
      ↓
login()
      ↓
Session created
```

Code:

```python
user = authenticate(username, password)

login(request, user)
```

Result:

```
request.user available
```

---

# 8. Logout System

Logout destroys session.

Code:

```python
logout(request)
```

Internally:

```
request.session.flush()
```

Result:

```
user becomes AnonymousUser
```

---

# 9. request.user

Every request me available hota hai.

### Logged in user

```
<User: shubham>
```

Check:

```python
request.user.is_authenticated
```

Result:

```
True
```

---

### Anonymous user

```
AnonymousUser
```

Result:

```
False
```

---

# 10. Login Protection

Decorator:

```python
@login_required
```

Purpose:

```
prevent anonymous access
```

Flow:

```
User access protected view
       ↓
check request.user.is_authenticated
       ↓
False → redirect login
True → allow access
```

---

# 11. Permissions

Permissions define **specific actions**.

Example:

```
add_course
change_course
delete_course
view_course
```

Check permission:

```python
request.user.has_perm("courses.add_course")
```

---

# 12. Groups & Roles

Groups implement **role based access control (RBAC)**.

Example roles:

```
Student
Instructor
Admin
```

Relationship:

```
User
  ↓
Group
  ↓
Permissions
```

Example:

```
Instructor
  add_course
  change_course
```

User assigned to group automatically inherits permissions.

---

# 13. Permission Check in View

Decorator:

```python
@permission_required("courses.add_course")
```

Manual check:

```python
request.user.has_perm("courses.add_course")
```

---

# 14. Built-in Authentication Views

Module:

```
django.contrib.auth.views
```

Important views:

```
LoginView
LogoutView
PasswordChangeView
PasswordResetView
PasswordResetConfirmView
```

Example:

```python
auth_views.LoginView.as_view()
```

Advantages:

```
secure
tested
less code
```

---

# 15. Password Reset System

Flow:

```
User clicks forgot password
        ↓
Enter email
        ↓
System sends reset link
        ↓
User opens link
        ↓
Enter new password
```

Reset URL format:

```
/reset/<uidb64>/<token>/
```

Example:

```
/reset/MQ/abc-token/
```

Token contains:

```
user id
timestamp
password state
secret key
```

Token invalid if:

```
password changed
token expired
user inactive
```

---

# 16. Password Validation

Django password rules:

```
minimum length
not common password
not numeric
not similar to username
```

Config:

```
AUTH_PASSWORD_VALIDATORS
```

---

# 17. Key Database Tables

Authentication related tables:

```
auth_user
auth_group
auth_permission
auth_user_groups
auth_group_permissions
django_session
```

Relationship:

```
User
 ↓
UserGroups
 ↓
Group
 ↓
Permissions
```

---

# 18. Full Authentication Flow

Complete lifecycle:

```
User register
       ↓
Password hashed
       ↓
User login
       ↓
authenticate()
       ↓
login()
       ↓
session created
       ↓
request.user available
       ↓
permissions checked
```

---

# 19. Project Features Implemented So Far

System currently supports:

```
Custom User Model
User Registration
Password Hashing
Login
Sessions
Logout
Login Protection
Permissions
Groups
Roles
Password Reset
Built-in Auth Views
```

---

# 20. One-Page Mental Model

Django authentication system simplified:

```
User Model
     ↓
Authentication Backend
     ↓
login()
     ↓
Session
     ↓
request.user
     ↓
Permissions
     ↓
Groups
     ↓
Authorization
```
