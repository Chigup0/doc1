# Security Implementation in TheDrive

TheDrive focuses on ensuring **user authentication** and **authorization** within the system, protecting sensitive information through token-based authentication and secure password hashing. While **end-to-end encryption (E2EE)** is not implemented at this stage, user input is filtered during login to prevent unauthorized access.

This document provides a detailed explanation of the **login system**, **token creation**, **user authentication** process, as well as the database schema for **storing user credentials** and **file system metadata**.

---

## 1. User Authentication

### 1.1 Password Hashing

To ensure that user passwords are stored securely, **TheDrive** uses **bcrypt** hashing for passwords. **bcrypt** is a widely adopted and secure password hashing algorithm that makes it difficult to reverse the hash into the original password.

#### Functions:
- **`verify_password(plain_password: str, hashed_password: str) -> bool`**: 
  This function compares the plain text password with the hashed password stored in the database.
  
- **`get_password_hash(password: str) -> str`**: 
  This function takes a plain password and hashes it using **bcrypt** before storing it in the database.

**Example Code**:
```
python
from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def verify_password(plain_password: str, hashed_password: str) -> bool:
    return pwd_context.verify(plain_password, hashed_password)

def get_password_hash(password: str) -> str:
    return pwd_context.hash(password)
```
1.2 Token Generation (JWT)

TheDrive uses JSON Web Tokens (JWT) for session management and user authentication. Once the user logs in, they are issued a JWT token, which is used for subsequent requests to authenticate the user.

Functions:

- ```create_access_token(data: dict, expires_delta: Optional[timedelta] = None)``` :
This function generates a JWT token, encoding user information (email) along with an expiration time. If no expiration is provided, it defaults to the value defined in the settings.

- ```get_token_from_cookie(request: Request) -> Optional[str]``` :
This function retrieves the JWT token from the cookies sent by the client. The token is stored in a cookie as part of the authentication flow.

Example Code:
```
from datetime import datetime, timedelta
from typing import Optional
from jose import JWTError, jwt
from fastapi import Request
from .config import settings

def create_access_token(data: dict, expires_delta: Optional[timedelta] = None):
    to_encode = data.copy()
    if expires_delta:
        expire = datetime.utcnow() + expires_delta
    else:
        expire = datetime.utcnow() + timedelta(minutes=settings.ACCESS_TOKEN_EXPIRE_MINUTES)
    to_encode.update({"exp": expire})
    encoded_jwt = jwt.encode(to_encode, settings.SECRET_KEY, algorithm="HS256")
    return encoded_jwt

def get_token_from_cookie(request: Request) -> Optional[str]:
    return request.cookies.get("access_token")
```

1.3 **User Authentication Flow**

Once a user logs in, a JWT token is created and returned in the response. This token is then stored in the client's cookies and used for authentication in subsequent requests.

User Login and Token Creation:

User enters their **email** and **password**.

The password is verified by comparing the plain password with the hashed password stored in the database.

If the credentials are valid, an access token (JWT) is generated and sent back to the client.

**Token Validation** :

For subsequent requests, the token is sent with the request (stored in the cookies).

The token is decoded using the **SECRET_KEY**.

If the token is valid and not expired, the user is authenticated, and their details are fetched from the database.

Example Code for Token Validation:
```
from fastapi import Depends, HTTPException, status
from sqlalchemy.orm import Session
from .database import get_db
from . import models

def get_current_user(token: str = Depends(get_token_from_cookie), db: Session = Depends(get_db)):
    if token is None:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Not authenticated")
    
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
    )
    try:
        payload = jwt.decode(token, settings.SECRET_KEY, algorithms=["HS256"])
        email: str = payload.get("sub")
        if email is None:
            raise credentials_exception
    except JWTError:
        raise credentials_exception
    user = db.query(models.User).filter(models.User.email == email).first()
    if user is None:
        raise credentials_exception
    return user
```

2. Database Schema

The User and FileSystemItem models are designed to store essential information about users and their uploaded files. These models are implemented using SQLAlchemy ORM and are used to persist data in the PostgreSQL database.

2.1 User Model

The User model is used to store user credentials (email and hashed password) and their associated files.

Fields:

```email``` : The unique email address used by the user for login.

```hashed_password``` : The hashed password of the user.

```items``` : The relationship between users and their FileSystemItem (files or folders).

User Model Example:

```
from sqlalchemy import Column, Integer, String
from sqlalchemy.orm import relationship
from .database import Base

class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True, index=True)
    email = Column(String, unique=True, index=True, nullable=False)
    hashed_password = Column(String, nullable=False)
    items = relationship("FileSystemItem", back_populates="owner")
```

2.2 FileSystemItem Model

The FileSystemItem model is used to store metadata about the files and folders uploaded by users. It tracks important file properties such as the file name, size, type, and S3 storage key.

Fields:

**name**: The name of the file or folder.

**type**: The type of the item (either a file or a folder).

**s3_key**: The key used to access the file stored in AWS S3.

**owner_id**: The ID of the user who owns the file or folder.

**parent_id**: The ID of the parent folder (if the item is a file within a folder).

**ai_processed**: Boolean flag indicating whether the file has been processed by the AI pipeline.

**ai_processing_status**: The current status of the AI processing (e.g., pending, completed).

**ai_entities**: The number of entities detected by the AI in the file.

**ai_relationships**: The number of relationships detected by the AI in the file.

**ai_communities**: The number of communities detected by the AI in the file.

FileSystemItem Model Example:
```
from sqlalchemy import Column, Integer, String, ForeignKey, Enum, Boolean
from sqlalchemy.orm import relationship
from .database import Base

class FileSystemItem(Base):
    __tablename__ = "filesystem_items"
    id = Column(String, primary_key=True, index=True)
    name = Column(String, index=True)
    type = Column(Enum("file", "folder", name="item_type_enum"), nullable=False)
    s3_key = Column(String, unique=True, nullable=True)
    mime_type = Column(String, nullable=True)
    size_bytes = Column(Integer, nullable=True)

    # AI processing fields
    ai_processed = Column(Boolean, default=False)
    ai_processing_status = Column(String, nullable=True, default="pending")
    ai_entities = Column(Integer, default=0)
    ai_relationships = Column(Integer, default=0)
    ai_communities = Column(Integer, default=0)
    
    owner_id = Column(Integer, ForeignKey("users.id"), nullable=False)
    parent_id = Column(String, ForeignKey("filesystem_items.id"), nullable=True)
    
    owner = relationship("User", back_populates="items")
    children = relationship("FileSystemItem", backref="parent", remote_side=[id], cascade="all, delete-orphan", single_parent=True)
```
3. Security Considerations
3.1 Token-Based Authentication

JWT tokens are used to authenticate users. The token is generated when a user logs in, and it is stored in the browser cookies. The token is sent with each request to validate the user’s identity.

JWT Generation: When a user successfully logs in, the system generates a JWT token, which includes the user’s email as a payload. This token expires after a certain time, and the user must log in again to get a new token.

Token Validation: Each time the user makes a request, the backend validates the token and retrieves the user’s details from the database.

3.2 Password Security

Passwords are hashed using bcrypt before being stored in the database, ensuring that plaintext passwords are never stored or exposed. bcrypt is a strong and widely-used hashing algorithm that incorporates salting and multiple rounds of hashing to make brute-force attacks difficult.

Conclusion

TheDrive implements a secure login system using JWT tokens for authentication and bcrypt for password hashing. The system stores user credentials and file metadata securely in the PostgreSQL database, with relationships between users and their files managed using SQLAlchemy ORM. The FileSystemItem model tracks file metadata and AI processing status, allowing the platform to offer intelligent file management and analysis features in the future.

This approach ensures that user data remains secure and private, while also providing a flexible foundation for adding advanced features such as AI-driven file analysis and semantic search.


---

### Explanation:
1. **Password Hashing**: Uses **bcrypt** for secure password hashing, making it hard for attackers to retrieve user passwords even if the database is compromised.
2. **JWT Tokens**: Tokens are used to manage user sessions. The token includes expiration and is validated for each request, ensuring that only authenticated users can access their data.
3. **Database Schema**: The **User** model stores user credentials, and the **FileSystemItem** model tracks file metadata, including the AI processing status and relationships.
4. **Security**: JWT-based authentication and bcrypt ensure secure password handling, while the database ensures integrity and safe storage of sensitive user data.

This Markdown file provides a **detailed overview** of the **security features**, focusing on **user authentication** and **data protection** within **TheDrive**.




