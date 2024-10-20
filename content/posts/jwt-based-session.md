---
title: "Secure JWT-based sessions"
description: "Security practices for JWT-based sessions when there is no other choice"
date: 2024-10-09T21:46:21+02:00
draft: true
showToc: true
TocOpen: false
tags: ["jwt", "web-security"]
UseHugoToc: true
---
## Introduction

Configuring website authentication using JWT-based sessions isn't the most common approach. However, sometimes due to the architecture of your solution, you might find that there is no other choice. This is especially true when your frontend cannot maintain traditional sessions via cookies and must rely on the backend's ability to handle both authentication (AuthN) and authorization (AuthZ). In this post, we'll explore the challenges associated with JWT-based sessions and discuss ways to mitigate or solve them, focusing on token expiration and revocation and addressing web security vulnerabilities.

> **Warning:** There's a considerable debate about using traditional database sessions versus JWTs. Both strategies have downsides, and you should evaluate them based on your solution's requirements.

## Understanding JWT, JWS, and JWE

Before diving into implementation details, it's essential to understand the basics about JWT:

- **JSON Web Token (JWT):** A compact, URL-safe means of representing claims to be transferred between two parties. It's widely used for authentication and information exchange. [RFC 7519](https://datatracker.ietf.org/doc/html/rfc7519).

- **JSON Web Signature (JWS):** A means of representing signed content using JSON data structures. A JWS is essentially a signed JWT. It ensures data integrity by allowing the recipient to verify that the data hasn't been altered. [RFC 7515](https://datatracker.ietf.org/doc/html/rfc7515).

- **JSON Web Encryption (JWE):** A means of representing encrypted content using JSON data structures. It provides confidentiality by encrypting the JWT so that only authorized parties can read the token's contents. [RFC 7516](https://datatracker.ietf.org/doc/html/rfc7516).

## Key Aspects of Session Management using JWT

Implementing JWTs as sessions requires careful consideration of token expiration and revocation, and the associated web security vulnerabilities.

### Token Expiration

The `expiration` field, `exp` claim, in a JWT, represents the time at which the token will expire. It is typically represented as a Unix timestamp and once the expiration time is reached, the token is considered invalid and should not be accepted for authentication or authorization purposes.

It is mandatory to validate the expiration time on the server side to ensure that expired tokens are not used.

> Adjusting your token expiration is a balancing act between user experience and security.

### Token Revocation

When using JWT-based sessions is important to know that each token ID (`jti` claim) needs to be stored in the backend's data layer to allow revocation. Implementing a robust token revocation strategy is crucial for maintaining security. Here's how to handle it:

The following actions should revoke a token:

  - **Logout**: Ensure tokens are invalidated when a user logs out.

  - **Explicit Revocation**: Provide an endpoint like `POST /api/v1/tokens/revoke` to allow users or services to revoke tokens explicitly.

  - **Password Changes and Resets**: Automatically revoke tokens if a user's password is changed or reset to prevent unauthorized access.


    > Implement a Time-To-Live (TTL) policy in your data layer for revoked and expired tokens to be removed. This prevents the database from becoming bloated with old tokens that need to be checked during verification.


### Addressing Web Security Vulnerabilities 

When using JWTs for session management, you must be vigilant about common web security vulnerabilities when storing the JWT in a browser cookie.

#### Cross-Site Scripting (XSS)

To mitigate XSS attacks you need to store them in cookies rather than local storage, as cookies can be secured with specific attributes:

- `HttpOnly`: Prevents JavaScript from accessing the cookies, reducing the risk of XSS attacks. [Learn more](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html#httponly-attribute).
- `Secure`: Ensures cookies are only sent over  HTTPS (TLS) connections. [Learn more](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html#secure-attribute).


#### Cross-Site Request Forgery (CSRF)

To protect against CSRF attacks follow the best practices and recommendations at OWASP [here](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html#token-based-mitigation).

- `SameSite`: Utilize the `SameSite=Lax` attribute in cookies to prevent leakage of information, preserving user privacy and providing some protection against cross-site request forgery attacks. It causes the browser to send the cookie in response to requests originating from and coming to the cookie's origin site (even if the user is coming from a different site). [Learn more](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html#samesite-attribute).

    > Even having a `Strict` directive, `SameSite`'s default `Lax` value provides a reasonable balance between security and usability. For more details on the SameSite values, check the following [section](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies#controlling_third-party_cookies_with_samesite).

#### Session Hijacking

To prevent session hijacking:

- Ensure that tokens can be revoked promptly in response to security events.
- Implement a token revocation procedure automating it as much as possible. Read more about it [here](#token-revocation).
- Mimic traditional session expiration behavior. [Read more](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html#session-expiration).
- Use HTTPS (TLS) connections all the time.

## Conclusion

While using JWTs for session management isn't the standard practice, it's sometimes necessary due to architectural constraints. By focusing on token expiration, token revocation, and addressing web security vulnerabilities, you can securely manage user sessions and protect your application from common security threats.