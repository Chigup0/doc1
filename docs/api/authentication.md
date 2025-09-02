# Authentication API for TheDrive

This document provides a detailed overview of the **Authentication API** implemented in **TheDrive** using **FastAPI**. The API includes endpoints for **signup**, **login**, **logout**, and retrieving **user details**. It uses **JWT tokens** for session management and **bcrypt** for password hashing.

---

## 1. Endpoints Overview

The Authentication API consists of the following endpoints:

1. **POST /signup**: Registers a new user.
2. **POST /login**: Authenticates a user and issues a JWT token.
3. **POST /logout**: Logs out the user by deleting the authentication cookie.
4. **GET /me**: Fetches the current logged-in user's details.

---

## 2. Endpoint Details

### 2.1 POST /signup

This endpoint allows a new user to register by providing an **email** and a **password**. It checks if the email is already registered in the system and, if not, hashes the password and stores the new user's details in the **PostgreSQL** database.

#### Request Body:
```
json
{
    "email": "user@example.com",
    "password": "strongpassword"
}
```

Response:

201 Created: User is successfully created.

409 Conflict: Email is already registered.

Example Response:

{
    "email": "user@example.com",
    "hashed_password": "$2b$12$...",
    "id": 1
}

Code Example:
@router.post("/signup", response_model=schemas.User, status_code=status.HTTP_201_CREATED)
def signup(user: schemas.UserCreate, db: Session = Depends(database.get_db)):
    db_user = get_user(db, email=user.email)
    if db_user:
        raise HTTPException(status_code=status.HTTP_409_CONFLICT, detail="Email already registered")
    hashed_password = get_password_hash(user.password)
    db_user = models.User(email=user.email, hashed_password=hashed_password)
    db.add(db_user)
    db.commit()
    db.refresh(db_user)
    return db_user

2.2 POST /login

This endpoint authenticates a user. It accepts the email and password via the OAuth2PasswordRequestForm and returns an access token (JWT) in a cookie for session management.

Request Body:
{
    "username": "user@example.com",
    "password": "strongpassword"
}

Response:

200 OK: Login successful, JWT token is issued and set in the access_token cookie.

401 Unauthorized: Incorrect email or password.

Example Response:

{
    "message": "Login successful"
}

Code Example:
```
@router.post("/login")
def login(response: Response, db: Session = Depends(database.get_db), form_data: OAuth2PasswordRequestForm = Depends()):
    user = get_user(db, email=form_data.username)
    if not user or not verify_password(form_data.password, user.hashed_password):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect email or password",
        )
    access_token = create_access_token(data={"sub": user.email})
    
    response.set_cookie(
        key="access_token", 
        value=access_token,  # UPDATED: No "Bearer " prefix
        httponly=True,
        samesite='lax',
    )
    return {"message": "Login successful"}
```
2.3 POST /logout

This endpoint logs out the current user by deleting the access_token cookie. It invalidates the user's session.

Request Body:

None

Response:

200 OK: Logout successful, access_token cookie is deleted.

Example Response:

{
    "message": "Logout successful"
}

Code Example:
```
@router.post("/logout")
def logout(response: Response):
    response.delete_cookie("access_token")
    return {"message": "Logout successful"}
```

2.4 GET /me

This endpoint fetches the details of the currently logged-in user. The access_token is retrieved from the cookie, and the user's identity is validated using the token. The user's details are returned if the token is valid.

**Response** :

**200 OK** : User details are returned.

**401 Unauthorized** : If the access_token is not provided or is invalid.

Example Response:
```
{
    "email": "user@example.com",
    "id": 1
}
```
Code Example:
```
@router.get("/me", response_model=schemas.User)
def read_users_me(current_user: models.User = Depends(get_current_user)):
    """
    Fetch the current logged in user by verifying the cookie.
    """
    return current_user


```

3. Security Features
3.1 Password Hashing

bcrypt is used for password hashing. When a user registers, their password is hashed and stored in the database. During login, the entered password is compared to the hashed version in the database to verify the credentials.

3.2 Token-Based Authentication

The system uses JWT tokens to manage user sessions. The JWT token is generated after successful login and stored in a secure, HTTP-only cookie to prevent XSS attacks. The token includes the user's email (as a claim) and an expiration time.

Token Expiration: The token expires after a pre-configured amount of time (e.g., 30 minutes), requiring the user to log in again.

3.3 Cookie Security

The JWT token is stored in an HTTP-only cookie, which makes it inaccessible to JavaScript running in the user's browser. This prevents XSS attacks by ensuring that malicious scripts cannot access the authentication token.

SameSite Cookie Attribute: The SameSite attribute is set to lax to provide protection against cross-site request forgery (CSRF) attacks.

3.4 Session Management

Login: When a user logs in, the backend generates a JWT token and stores it in the user's cookies.

Logout: The /logout endpoint removes the access_token cookie, effectively ending the user's session.

User Data: The /me endpoint uses the token to retrieve the current user's data from the database.

4. Data Flow for Authentication
4.1 Sign-up Flow

User enters their email and password.

The backend checks if the email already exists in the database.

If the email is not registered, the backend hashes the password and stores the user details in the database.

The backend returns the user data (without the password) as a response.

4.2 Login Flow

User enters their email and password.

The backend checks if the provided credentials are correct by verifying the password hash.

If valid, the backend generates a JWT token and sends it to the user in an HTTP-only cookie.

The user can now access protected endpoints by sending the token with subsequent requests.

4.3 Logout Flow

User clicks on the Logout button, which triggers the /logout endpoint.

The backend deletes the access_token cookie.

The user is logged out and must log in again to obtain a new token.

4.4 Fetch User Data Flow

User makes a GET /me request.

The backend retrieves the access_token from the user's cookies.

The backend decodes the token to verify the user's identity.

The user's data is returned in the response.

5. Conclusion

TheDrive’s Authentication API ensures that users can securely sign up, log in, and manage their sessions with JWT tokens stored in cookies. Passwords are hashed using bcrypt, ensuring that sensitive data is not exposed. The JWT-based authentication system provides secure session management, while cookie security features like HTTP-only and SameSite protect against common web security threats such as XSS and CSRF.

This authentication system enables secure and efficient user management for TheDrive, with easy-to-understand data flows for logging in, logging out, and fetching user details.


---

### **Explanation**:
- The **Authentication API** consists of endpoints for **signup**, **login**, **logout**, and **user details** retrieval.
- **JWT tokens** are used for session management, and **bcrypt** ensures **secure password storage**.
- **Cookie security** with **HTTP-only** and **SameSite** attributes protects against common web vulnerabilities.