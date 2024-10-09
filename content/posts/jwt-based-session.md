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

Configuring website authentication using JWT-based sessions isn't the most common approach. However, sometimes due to the architecture of your solution, you might find that there is no other choice. This is especially true when your frontend cannot maintain traditional sessions via cookies or interact with a database, and must rely on the backend's ability to issue tokens. In this article, we'll explore the challenges associated with JWT-based sessions and discuss ways to mitigate or solve them, focusing on token expiration, token revocation, and addressing web security vulnerabilities.

## Understanding JWT, JWS, and JWE

Before diving into implementation details, it's essential to understand the basics:

- **JWT (JSON Web Token):** A compact, URL-safe means of representing claims to be transferred between two parties. It's widely used for authentication and information exchange. [Learn more on Wikipedia](https://en.wikipedia.org/wiki/JSON_Web_Token).

- **JWS (JSON Web Signature):** A means of representing signed content using JSON data structures. A JWS is essentially a signed JWT. It ensures data integrity by allowing the recipient to verify that the data hasn't been altered. [Learn more on Wikipedia](https://en.wikipedia.org/wiki/JSON_Web_Signature).

- **JWE (JSON Web Encryption):** A means of representing encrypted content using JSON data structures. It provides confidentiality by encrypting the JWT so that only authorized parties can read the token's contents. [Learn more on Wikipedia](https://en.wikipedia.org/wiki/JSON_Web_Encryption).

## Key Aspects of JWT Session Management

Implementing JWTs as sessions requires careful consideration of token expiration, token revocation, and web security vulnerabilities.

### Token Expiration

Adjusting your token expiration (`exp` claim) is a balancing act between user experience and security. Here are some industry-standard durations:

- **Access Tokens (`access_token`):** Typically valid for **1 day**. These tokens grant access to protected resources and should have a shorter lifespan to minimize the risk if compromised.

- **Refresh Tokens (`refresh_token`):** Generally valid for **15 days**. They allow the user to obtain new access tokens without re-authenticating.

**Considerations:**

- **User Experience vs. Security:** Shorter expirations enhance security but may inconvenience users due to frequent logins.

- **Application Needs:** Adjust token lifespans based on the sensitivity of the protected resources and the typical usage patterns of your users.

### Token Revocation

Implementing a robust token revocation strategy is crucial for maintaining security. Here's how to handle it:

- **Revocation Events:** The following actions should revoke both `access_token` and `refresh_token`:

  - **Application Logout:** Ensure tokens are invalidated when a user logs out.

  - **Explicit Revocation:** Provide an endpoint like `POST /api/v1/tokens/revoke` to allow users or services to revoke tokens explicitly.

  - **Password Changes and Resets:** Automatically revoke tokens if a user's password is changed or reset to prevent unauthorized access.

- **Database Cleanup:** Implement a Time-To-Live (TTL) policy in your database for revoked tokens. This prevents the database from becoming bloated with old tokens that need to be checked during verification.

  - **Performance Consideration:** Regularly clean up expired tokens to maintain optimal performance.

> **Warning:** There's considerable debate about using traditional database sessions versus JWTs. Both strategies have downsides, and you should evaluate them based on your solution's requirements.

### Addressing Web Security Vulnerabilities

When using JWTs for session management, you must be vigilant about common web security vulnerabilities.

#### Cross-Site Scripting (XSS)

To mitigate XSS attacks:

- **Store JWTs in Cookies:** Use cookies to store JWTs rather than local storage, as cookies can be secured with specific attributes.

- **Secure Cookie Attributes:**

  - **`HttpOnly`:** Prevents JavaScript from accessing the cookies, reducing the risk of XSS attacks. [Learn more](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html#httponly-attribute).

  - **`Secure`:** Ensures cookies are only sent over HTTPS connections. [Learn more](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html#secure-attribute).

- **Cookie Size Limitation:** Be mindful of the size of your JWTs when storing them in cookies. Browsers limit the size of cookies to approximately **4 KiB**. Large tokens may not fit, leading to potential issues with authentication.

#### Cross-Site Request Forgery (CSRF)

To protect against CSRF attacks:

- **Implement CSRF Tokens:** Use anti-CSRF tokens in forms and verify them on the server side.

- **Double Submit Cookie Pattern:** Consider implementing the double submit cookie pattern where a CSRF token is stored both in a cookie and as a request parameter.

- **SameSite Cookies:** Utilize the `SameSite=Lax` attribute in cookies to restrict cross-origin requests. This helps prevent CSRF attacks by not sending cookies with cross-site requests. [Learn more](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html#samesite-attribute).

#### Session Hijacking

To prevent session hijacking:

- **Token Revocation:** Ensure that tokens can be revoked promptly in response to security events.

- **Logout Endpoint:** Implement a `POST /api/v1/tokens/revoke` endpoint to revoke tokens during logout, mimicking traditional session expiration behavior. [Read more](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html#session-expiration).

- **Use HTTPS Everywhere:** Ensure all communication between the client and server is encrypted to prevent man-in-the-middle attacks.

## Conclusion

While using JWTs for session management isn't the standard practice, it's sometimes necessary due to architectural constraints. By focusing on token expiration, token revocation, and addressing web security vulnerabilities, you can securely manage user sessions and protect your application from common security threats.

---

Remember, the key is to weigh the pros and cons of JWT-based sessions against traditional methods and choose the one that best fits your application's needs. Always stay informed about the latest security practices to keep your application and users safe.