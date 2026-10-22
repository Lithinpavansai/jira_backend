\# Write-up #1: Why my API said "Invalid email or password" to a request with no password



\*\*Project:\*\* KIET Jira backend (FastAPI, python-jose JWT, PostgreSQL) · \*\*Date:\*\* 22 Oct 2026



\## Symptom

A call to `GET /auth/me` with a valid Bearer token returned `200`. About a minute later, the same call with the same token returned `401`:



```json

{"success":false,"message":"Invalid email or password","error\_code":"UNAUTHORIZED","path":"/auth/me","details":null}

```

The request contained no email and no password. The server log showed only `GET /auth/me 401 Unauthorized`, with no reason.



\## How I reproduced it

1\. Started the server with `ACCESS\_TOKEN\_EXPIRE\_MINUTES=1` (the default in `.env` is 30).

2\. Logged in as the admin and saved the token.

3\. Called `/auth/me` right away: `200`.

4\. Waited 65 seconds and called it again with the same token: `401`.



\## Root cause

\- python-jose raises `ExpiredSignatureError` for an expired token. It is a subclass of `JWTError`.

\- `app/auth/dependencies.py` catches the broad `JWTError` and calls `raise\_unauthorized()` with no message.

\- `raise\_unauthorized()` in `app/exceptions.py` defaults to `ErrorMessages.INVALID\_CREDENTIALS`, which is the text meant for failed logins.

\- Result: expired, tampered and malformed tokens all return the same 401 and the same login-failure message.



\## Why it matters

A client cannot tell "log in again" from "wrong credentials". Users reset passwords that were never wrong, and support spends time on the wrong problem.



\## Proposed fix (not yet applied)

Catch the specific error first, because it is a subclass of the broader one:

```python

from jose import jwt, JWTError, ExpiredSignatureError



&#x20;   except ExpiredSignatureError:

&#x20;       raise\_unauthorized("Token expired. Please log in again.")

&#x20;   except JWTError:

&#x20;       raise\_unauthorized("Invalid token.")

```

Also log the reason server-side. A separate `TOKEN\_EXPIRED` error code would help clients, since `raise\_unauthorized` currently always sends `UNAUTHORIZED`.



\## How a support engineer can tell the difference today

A JWT payload is base64 text. Decode the middle part of the token and read the `exp` field (a Unix timestamp). If it is earlier than the time in the response's `date` header, the token expired and the user only needs to log in again.



\## What I would tell the customer

"Your session has expired. Please log in again to get a new token. You do not need to reset your password."

