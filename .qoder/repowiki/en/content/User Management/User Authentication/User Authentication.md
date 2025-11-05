# User Authentication

<cite>
**Referenced Files in This Document**   
- [interface.go](file://services/auth/interface.go)
- [signin.go](file://services/auth/signin.go)
- [auth.go](file://models/auth/auth_token.go)
- [session.go](file://models/auth/session.go)
- [source.go](file://models/auth/source.go)
- [common.go](file://modules/auth/common.go)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [Authentication Flow Overview](#authentication-flow-overview)
3. [Authentication Interface and Implementation](#authentication-interface-and-implementation)
4. [Form-Based and API Authentication](#form-based-and-api-authentication)
5. [Session Management and Middleware](#session-management-and-middleware)
6. [Authentication Sources](#authentication-sources)
7. [Common Authentication Issues](#common-authentication-issues)
8. [Performance Considerations](#performance-considerations)
9. [Security Best Practices](#security-best-practices)

## Introduction
Gitea provides a robust and extensible user authentication system that supports multiple authentication methods including database, LDAP, OAuth2, PAM, and SMTP. The system is designed to handle both form-based authentication for web users and token-based authentication for API clients. This document details the authentication flow from HTTP request to session creation, covering the core interfaces, implementations, and security considerations.

## Authentication Flow Overview

```mermaid
flowchart TD
A[HTTP Request] --> B{Authentication Method}
B --> C[Form-Based Login]
B --> D[API Token Authentication]
B --> E[OAuth2 Authentication]
B --> F[LDAP Authentication]
C --> G[Validate Credentials]
D --> H[Validate Access Token]
E --> I[Validate OAuth2 Token]
F --> J[Validate LDAP Credentials]
G --> K{Valid?}
H --> K
I --> K
J --> K
K --> |Yes| L[Create Session]
K --> |No| M[Return Error]
L --> N[Set Session Cookie]
N --> O[Authenticated User]
style K fill:#f9f,stroke:#333
style L fill:#bbf,stroke:#333
```

**Diagram sources**
- [signin.go](file://services/auth/signin.go)
- [auth.go](file://models/auth/auth_token.go)

**Section sources**
- [signin.go](file://services/auth/signin.go)
- [auth.go](file://models/auth/auth_token.go)

## Authentication Interface and Implementation

The authentication system in Gitea is built around a pluggable interface defined in `services/auth/interface.go`. This interface allows for multiple authentication methods to be implemented and used simultaneously.

```mermaid
classDiagram
class Method {
+Verify(request *http.Request, w http.ResponseWriter, store DataStore, sess SessionStore) (*User, error)
+Name() string
}
class PasswordAuthenticator {
+Authenticate(ctx context.Context, user *User, login, password string) (*User, error)
}
class SynchronizableSource {
+Sync(ctx context.Context, updateExisting bool) error
}
class SessionStore {
// Implementation of session.Store
}
Method <|-- LDAPAuth
Method <|-- OAuth2Auth
Method <|-- DBAuth
Method <|-- SMTPAuth
Method <|-- PAMAuth
Method <|-- SSPIAuth
PasswordAuthenticator <|-- LDAPAuth
PasswordAuthenticator <|-- DBAuth
PasswordAuthenticator <|-- SMTPAuth
PasswordAuthenticator <|-- PAMAuth
SynchronizableSource <|-- LDAPAuth
SynchronizableSource <|-- OAuth2Auth
note right of Method
Core interface for all authentication methods
Verify() handles the authentication logic
end
note right of PasswordAuthenticator
Interface for password-based authentication
Used by DB, LDAP, SMTP, and PAM methods
end
```

**Diagram sources**
- [interface.go](file://services/auth/interface.go)
- [source.go](file://models/auth/source.go)

**Section sources**
- [interface.go](file://services/auth/interface.go)
- [source.go](file://models/auth/source.go)

## Form-Based and API Authentication

Gitea supports both form-based authentication for web interfaces and token-based authentication for API clients. The system uses a unified authentication pipeline that can handle both types of requests.

```mermaid
sequenceDiagram
participant Client as "Web Client"
participant Router as "Web Router"
participant Auth as "AuthService"
participant Source as "AuthSource"
participant DB as "Database"
participant Session as "SessionStore"
Client->>Router : POST /user/login
Router->>Auth : ParseForm()
Auth->>Auth : ValidateForm()
Auth->>Source : VerifyCredentials(username, password)
Source->>DB : QueryUser(username)
DB-->>Source : UserRecord
Source->>Source : CheckPasswordHash(password)
Source-->>Auth : AuthenticationResult
Auth->>Session : CreateSession(user)
Session-->>Auth : SessionID
Auth->>Router : SetCookie(sessionID)
Router-->>Client : RedirectToDashboard
Note over Client,Router : Form-based authentication flow
```

```mermaid
sequenceDiagram
participant Client as "API Client"
participant Router as "API Router"
participant Auth as "AuthTokenService"
participant DB as "Database"
participant Session as "SessionStore"
Client->>Router : GET /api/v1/user<br/>Authorization : token abc123
Router->>Auth : ExtractToken()
Auth->>DB : FindToken(abc123)
DB-->>Auth : TokenRecord
Auth->>Auth : CheckTokenExpiration()
Auth->>DB : GetUser(token.UserID)
DB-->>Auth : UserRecord
Auth->>Session : CreateAPISession(user)
Session-->>Auth : SessionContext
Auth-->>Router : AttachUserContext()
Router-->>Client : ReturnUserData
Note over Client,Router : API token authentication flow
```

**Diagram sources**
- [signin.go](file://services/auth/signin.go)
- [auth.go](file://models/auth/auth_token.go)

**Section sources**
- [signin.go](file://services/auth/signin.go)
- [auth.go](file://models/auth/auth_token.go)

## Session Management and Middleware

The session management system in Gitea is responsible for maintaining user sessions after successful authentication. The `auth.Middleware` component plays a crucial role in authenticating users on each request.

```mermaid
flowchart TD
A[Incoming Request] --> B{Has Session Cookie?}
B --> |Yes| C[Validate Session ID]
B --> |No| D{Has Authorization Header?}
D --> |Yes| E[Validate API Token]
D --> |No| F[Anonymous Access]
C --> G{Session Valid?}
E --> H{Token Valid?}
G --> |Yes| I[Load User Data]
H --> |Yes| I
G --> |No| J[Clear Invalid Session]
H --> |No| K[Return Unauthorized]
I --> L[Attach User to Context]
L --> M[Proceed to Handler]
J --> N[Redirect to Login]
K --> O[Return 401]
style G fill:#f9f,stroke:#333
style H fill:#f9f,stroke:#333
style I fill:#bbf,stroke:#333
```

**Diagram sources**
- [session.go](file://models/auth/session.go)
- [common.go](file://modules/auth/common.go)

**Section sources**
- [session.go](file://models/auth/session.go)
- [common.go](file://modules/auth/common.go)

## Authentication Sources

Gitea supports multiple authentication sources, each implementing the common authentication interface. The `Source` type in `models/auth/source.go` defines the structure for authentication sources.

```mermaid
classDiagram
class Source {
+ID int64
+Type Type
+Name string
+IsActive bool
+IsSyncEnabled bool
+TwoFactorPolicy string
+Cfg Config
+CreatedUnix TimeStamp
+UpdatedUnix TimeStamp
}
class Config {
<<interface>>
+SetAuthSource(*Source)
}
class LDAPConfig {
+BindDN string
+BindPassword string
+UserSearchBase string
+UserFilter string
+GroupSearchBase string
}
class OAuth2Config {
+ClientID string
+ClientSecret string
+TokenURL string
+AuthURL string
+ProfileURL string
+Scopes string
}
class SMTPConfig {
+Host string
+Port int
+SecurityProtocol int
+SkipVerify bool
}
class PAMConfig {
+ServiceName string
+EmailDomain string
}
Source --> Config : "has"
Config <|-- LDAPConfig
Config <|-- OAuth2Config
Config <|-- SMTPConfig
Config <|-- PAMConfig
note right of Source
Base structure for all authentication sources
Type determines the authentication method
end
note right of Config
Interface for authentication-specific configuration
Implemented by each authentication method
end
```

**Diagram sources**
- [source.go](file://models/auth/source.go)
- [interface.go](file://services/auth/interface.go)

**Section sources**
- [source.go](file://models/auth/source.go)
- [interface.go](file://services/auth/interface.go)

## Common Authentication Issues

This section addresses common issues encountered during the authentication process, including failed logins, account lockouts, and session expiration.

```mermaid
flowchart TD
A[Authentication Issue] --> B{Issue Type}
B --> C[Failed Login]
C --> C1[Check Credentials]
C1 --> C2{Valid?}
C2 --> |No| C3[Return Error]
C2 --> |Yes| C4[Check Account Status]
C4 --> C5{Active?}
C5 --> |No| C6[Account Inactive]
C5 --> |Yes| C7[Check 2FA]
B --> D[Account Lockout]
D --> D1[Check Failed Attempts]
D1 --> D2{Exceeded Limit?}
D2 --> |Yes| D3[Lock Account]
D2 --> |No| D4[Allow Retry]
B --> E[Session Expiration]
E --> E1[Check Session Age]
E1 --> E2{Expired?}
E2 --> |Yes| E3[Clear Session]
E2 --> |No| E4[Extend Session]
style C2 fill:#f9f,stroke:#333
style D2 fill:#f9f,stroke:#333
style E2 fill:#f9f,stroke:#333
```

**Section sources**
- [signin.go](file://services/auth/signin.go)
- [session.go](file://models/auth/session.go)

## Performance Considerations

When handling authentication under high load, several performance considerations must be addressed to maintain system responsiveness and security.

```mermaid
graph TD
A[High Load Authentication] --> B[Connection Pooling]
A --> C[Caching Strategies]
A --> D[Rate Limiting]
A --> E[Asynchronous Operations]
B --> B1[Database Connections]
B --> B2[LDAP Connections]
B --> B3[SMTP Connections]
C --> C1[Session Caching]
C --> C2[User Data Caching]
C --> C3[Authentication Result Caching]
D --> D1[Login Attempt Limits]
D --> D2[IP-Based Rate Limiting]
D --> D3[Account Lockout Policies]
E --> E1[Background Sync Operations]
E --> E2[Async Email Verification]
E --> E3[Deferred Logging]
style B fill:#bbf,stroke:#333
style C fill:#bbf,stroke:#333
style D fill:#bbf,stroke:#333
style E fill:#bbf,stroke:#333
```

**Section sources**
- [common.go](file://modules/auth/common.go)
- [session.go](file://models/auth/session.go)

## Security Best Practices

This section outlines security best practices for protecting credentials during transmission and storage in the Gitea authentication system.

```mermaid
flowchart TD
A[Security Best Practices] --> B[Transmission Security]
A --> C[Storage Security]
A --> D[Authentication Security]
A --> E[Session Security]
B --> B1[Use HTTPS/TLS]
B --> B2[Secure Cookies]
B --> B3[HSTS Headers]
C --> C1[Hash Passwords with Bcrypt]
C --> C2[Encrypt Sensitive Data]
C --> C3[Secure Configuration Storage]
D --> D1[Rate Limiting]
D --> D2[Account Lockout]
D --> D3[Multi-Factor Authentication]
E --> E1[Session Expiration]
E --> E2[Secure Session IDs]
E --> E3[Session Regeneration]
style B fill:#bbf,stroke:#333
style C fill:#bbf,stroke:#333
style D fill:#bbf,stroke:#333
style E fill:#bbf,stroke:#333
```

**Section sources**
- [common.go](file://modules/auth/common.go)
- [session.go](file://models/auth/session.go)
- [source.go](file://models/auth/source.go)