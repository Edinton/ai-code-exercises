# API Documentation Exercise — User Registration Endpoint


**Exercise:** API Documentation

**Endpoint chosen:** `POST /api/users/register` (Python/Flask example)

---

## 1. Original API Endpoint Code

```python
@app.route('/api/users/register', methods=['POST'])
def register_user():
    """Register a new user"""
    data = request.get_json()
    # Validate required fields
    required_fields = ['username', 'email', 'password']
    for field in required_fields:
        if field not in data:
            return jsonify({
                'error': 'Missing required field',
                'message': f'{field} is required'
            }), 400
    # Check if username or email already exists
    if User.query.filter_by(username=data['username']).first():
        return jsonify({
            'error': 'Username taken',
            'message': 'Username is already in use'
        }), 409
    if User.query.filter_by(email=data['email']).first():
        return jsonify({
            'error': 'Email exists',
            'message': 'An account with this email already exists'
        }), 409
    # Validate email format
    if not re.match(r"^[^@]+@[^@]+\.[^@]+$", data['email']):
        return jsonify({
            'error': 'Invalid email',
            'message': 'Please provide a valid email address'
        }), 400
    # Validate password strength
    if len(data['password']) < 8:
        return jsonify({
            'error': 'Weak password',
            'message': 'Password must be at least 8 characters long'
        }), 400
    # Create new user
    try:
        password_hash = generate_password_hash(data['password'])
        new_user = User(
            username=data['username'],
            email=data['email'].lower(),
            password_hash=password_hash,
            created_at=datetime.utcnow(),
            role='user'
        )
        db.session.add(new_user)
        db.session.commit()
        confirmation_token = generate_confirmation_token(new_user.id)
        try:
            send_confirmation_email(new_user.email, confirmation_token)
        except Exception as e:
            app.logger.error(f"Failed to send confirmation email: {str(e)}")
        user_data = {
            'id': new_user.id,
            'username': new_user.username,
            'email': new_user.email,
            'created_at': new_user.created_at.isoformat(),
            'role': new_user.role
        }
        return jsonify({
            'message': 'User registered successfully',
            'user': user_data
        }), 201
    except Exception as e:
        db.session.rollback()
        app.logger.error(f"Error registering user: {str(e)}")
        return jsonify({
            'error': 'Server error',
            'message': 'Failed to register user'
        }), 500
```

---

## 2. Comprehensive Endpoint Documentation (Prompt 1)

### `POST /api/users/register`

**Purpose:** Registers a new user account. Validates required fields, checks for duplicate username/email, validates email format and password strength, creates the user record with a hashed password, and triggers a best-effort confirmation email.

**Authentication:** None required — public, unauthenticated endpoint (registration must be reachable before a user has credentials).

**Request**

*Headers:*
| Header | Required | Description |
|---|---|---|
| `Content-Type` | Yes | Must be `application/json` |

*Body parameters (JSON):*
| Field | Type | Required | Description |
|---|---|---|---|
| `username` | string | Yes | Desired username; must be unique |
| `email` | string | Yes | Valid email address; must be unique (case-insensitive, stored lowercased) |
| `password` | string | Yes | Plaintext password; minimum 8 characters, hashed before storage |

**Response — Success `201 Created`**
```json
{
  "message": "User registered successfully",
  "user": {
    "id": 42,
    "username": "jdoe",
    "email": "jdoe@example.com",
    "created_at": "2026-07-13T10:15:30.123456",
    "role": "user"
  }
}
```
`password`/`password_hash` is never included in the response.

**Error responses:**

| Status | Error code | When it occurs |
|---|---|---|
| `400` | `Missing required field` | `username`, `email`, or `password` absent from body |
| `400` | `Invalid email` | Email fails regex format check |
| `400` | `Weak password` | Password under 8 characters |
| `409` | `Username taken` | Username already exists |
| `409` | `Email exists` | Email already exists |
| `500` | `Server error` | Unhandled exception during user creation (DB write failure, etc.) |

**Example requests:**

*Example 1 — successful registration:*
```
POST /api/users/register
Content-Type: application/json

{
  "username": "jdoe",
  "email": "JDoe@Example.com",
  "password": "SecurePass123"
}
```
```json
// 201 Created
{
  "message": "User registered successfully",
  "user": {
    "id": 42,
    "username": "jdoe",
    "email": "jdoe@example.com",
    "created_at": "2026-07-13T10:15:30.123456",
    "role": "user"
  }
}
```

*Example 2 — duplicate email:*
```
POST /api/users/register
Content-Type: application/json

{
  "username": "newuser",
  "email": "jdoe@example.com",
  "password": "AnotherPass123"
}
```
```json
// 409 Conflict
{
  "error": "Email exists",
  "message": "An account with this email already exists"
}
```

**Rate limiting / special considerations:**
- No rate limiting visible in this code — a real gap, since public registration endpoints are common spam/abuse targets.
- Confirmation email failures are logged but **do not fail the request** — the user is still created and receives `201` even if the email never sends.
- Field validation happens in a fixed order (missing fields → duplicates → format → strength), so error responses only ever report the *first* problem found, not all problems at once.

---

## 3. Converted Documentation Format — OpenAPI 3.0 (Prompt 2)

```yaml
openapi: 3.0.3
info:
  title: User Registration API
  version: "1.0.0"
  description: Endpoint for registering new user accounts.
paths:
  /api/users/register:
    post:
      summary: Register a new user
      operationId: registerUser
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [username, email, password]
              properties:
                username:
                  type: string
                  description: Desired username; must be unique
                email:
                  type: string
                  format: email
                  description: Valid, unique email address (stored lowercased)
                password:
                  type: string
                  minLength: 8
                  description: Plaintext password; hashed before storage
      responses:
        '201':
          description: User registered successfully
          content:
            application/json:
              schema:
                type: object
                properties:
                  message:
                    type: string
                    example: User registered successfully
                  user:
                    type: object
                    properties:
                      id:
                        type: integer
                        example: 42
                      username:
                        type: string
                        example: jdoe
                      email:
                        type: string
                        example: jdoe@example.com
                      created_at:
                        type: string
                        format: date-time
                      role:
                        type: string
                        example: user
        '400':
          description: Validation error (missing field, invalid email, or weak password)
          content:
            application/json:
              schema:
                type: object
                properties:
                  error:
                    type: string
                    enum: [Missing required field, Invalid email, Weak password]
                  message:
                    type: string
              examples:
                missingField:
                  value:
                    error: Missing required field
                    message: "username is required"
                invalidEmail:
                  value:
                    error: Invalid email
                    message: Please provide a valid email address
                weakPassword:
                  value:
                    error: Weak password
                    message: Password must be at least 8 characters long
        '409':
          description: Username or email already exists
          content:
            application/json:
              schema:
                type: object
                properties:
                  error:
                    type: string
                    enum: [Username taken, Email exists]
                  message:
                    type: string
              examples:
                usernameTaken:
                  value:
                    error: Username taken
                    message: Username is already in use
                emailExists:
                  value:
                    error: Email exists
                    message: An account with this email already exists
        '500':
          description: Unexpected server error during registration
          content:
            application/json:
              schema:
                type: object
                properties:
                  error:
                    type: string
                    example: Server error
                  message:
                    type: string
                    example: Failed to register user
      security: []
```

---

## 4. Developer Usage Guide (Prompt 3)

**Target audience:** Junior developers integrating this registration endpoint into a frontend app for the first time
**Tone:** Friendly, practical

### Using the User Registration Endpoint — A Quick Guide

**1. Authentication**
No auth token needed — registration has to work *before* someone has an account, so `POST /api/users/register` is wide open.

**2. Formatting your request**
Send a `POST` with a JSON body and don't forget the `Content-Type: application/json` header:

```javascript
const response = await fetch("https://your-api.example.com/api/users/register", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    username: "jdoe",
    email: "jdoe@example.com",
    password: "SecurePass123"
  })
});
const data = await response.json();
```

All three fields are required — leaving any one out gets you a `400` immediately, before the server even checks if the username/email is taken.

**3. Handling the response**
On success (`201`), you get back the new user's public info — no password, ever:
```json
{ "message": "User registered successfully", "user": { "id": 42, "username": "jdoe" } }
```
Grab `user.id` right away if you need it (e.g. to log the user in or redirect to onboarding).

**4. Dealing with common errors**

```javascript
if (!response.ok) {
  switch (data.error) {
    case "Username taken":
    case "Email exists":
      // Show inline "already registered" message next to the relevant field
      break;
    case "Invalid email":
    case "Weak password":
    case "Missing required field":
      // Client-side validation slipped through -- show the message field to the user
      break;
    default:
      // 500 or unexpected -- show a generic "something went wrong" message
  }
}
```
The server checks in a fixed order (missing fields → duplicate username/email → email format → password strength), so if a user's form has *multiple* problems, you'll only see one error at a time — consider validating on the client side too, so users see all issues before submitting.

**5. Example code (JavaScript, using fetch)**

```javascript
async function registerUser(username, email, password) {
  const res = await fetch("/api/users/register", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ username, email, password })
  });
  const data = await res.json();

  if (res.ok) {
    console.log("Registered:", data.user);
    return data.user;
  } else {
    console.error(`${data.error}: ${data.message}`);
    throw new Error(data.message);
  }
}
```

**Heads-up:** a successful `201` doesn't guarantee the confirmation email actually sent — the server logs email failures but still returns success. If your app relies on email confirmation for onboarding, consider adding a "resend confirmation" option in your UI just in case.
