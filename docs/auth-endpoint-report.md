# Authentication Endpoint Report

## Endpoint Details
- **Path:** `/api/v1/auth/token`
- **Method:** `POST`
- **File:** `src/qData/api/v1/auth.py`

## Credentials Required
The endpoint expects a JSON payload containing:
- `username`: A string representing the username.
- `password`: A string representing the user's password.

```json
{
  "username": "example_user",
  "password": "example_password"
}
```

## Current Implementation Status
The system uses **Supabase** for user authentication and management. The endpoint queries the Supabase `users` table via the `user_store.py` module. 

A user must satisfy the following conditions to successfully authenticate:
1. Exist in the Supabase `users` table.
2. Have `is_active` set to `True`.
3. Have `service_type = rest` in the current implementation.
4. Provide a plaintext password that matches the `hashed_password` stored in the database (verified via `bcrypt`).

### Note on Generating Tokens
The access token generated is signed with `REST_JWT_SECRET`. In the current implementation, this same REST token is also accepted by the gRPC interceptor.

## Response
On successful authentication, it returns a JSON object containing the JWT token and its type:
```json
{
  "access_token": "eyJ0eXAiOi...",
  "token_type": "bearer"
}
```

If the credentials are incorrect or the user is inactive/unauthorized for the service, it returns:
- **Status Code:** `400 Bad Request`
- **Body:** `{"detail": "Incorrect username or password"}`

