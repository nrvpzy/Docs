## Security: SSO + RBAC — Concise Memory Version

Remember this:

```text
ADFS authenticates → JWT proves session → toolboxadmin authorizes → UI/backend enforce access
```

Or:

```text
ADFS = Who are you?
JWT = Proof you are logged in
toolboxadmin = What can you do?
UI/services = Enforce allowed actions
```

---

# 1. Authentication Flow — SSO

## Step-by-step

```text
User opens toolboxui
   ↓
UI checks if valid session/JWT cookie exists
   ↓
No session → redirect to Login with SSO
   ↓
User clicks SSO login
   ↓
Browser goes to toolboxadfsbridge
   ↓
toolboxadfsbridge redirects to ADFS/JLR SSO
   ↓
ADFS authenticates user using enterprise credentials/MFA
   ↓
ADFS redirects back to toolboxadfsbridge with authorization code
   ↓
toolboxadfsbridge exchanges code for tokens
   ↓
Gets ID token/access token + OIDC user claims
   ↓
JWT/access token stored in secure cookie/session
   ↓
User redirected back to toolboxui
   ↓
UI loads protected app
```

---

# 2. What Each Component Did

## `toolboxui`

- Shows login page
- Redirects user to SSO
- After login, calls APIs with JWT cookie
- Shows/hides pages/buttons based on permissions

## `toolboxadfsbridge`

- OAuth2/OIDC client
- Starts login with ADFS
- Handles callback from ADFS
- Extracts OIDC user claims
- Establishes app session/token cookie

## ADFS / JLR SSO

- Enterprise identity provider
- Authenticates user
- Enforces corporate login/MFA policies
- Issues tokens

## Microservices

- OAuth2 Resource Servers
- Validate JWT on every request
- Use `cookieTokenResolver()` to read token from cookie
- Use `JwtConverter` to convert JWT claims into Spring Security principal

---

# 3. After Login — How User Stays Authenticated

```text
JWT/access token is stored in secure cookie
   ↓
Browser automatically sends cookie with API calls
   ↓
Each microservice extracts JWT from cookie
   ↓
Service validates signature, issuer, expiry
   ↓
If valid → request allowed to reach controller
   ↓
If invalid/expired → 401
```

Key phrase:

> Each service independently validated the JWT, so we did not rely only on frontend or gateway security.

---

# 4. Authorization Flow — RBAC

Authentication only says:

```text
User is valid
```

Authorization says:

```text
What is user allowed to do?
```

RBAC was handled by `toolboxadmin`.

Model:

```text
User → Roles → Authorization Codes
```

Example authorization codes:

```text
VIEW_ORDER
SAVE_ORDER
MODIFY_ORDER
RETURN_VEHICLE.PUT
CANCEL_ORDER.PUT
VIEW_USER_MANAGEMENT
VIEW_ROLE_MANAGEMENT
```

---

# 5. RBAC Flow After Login

```text
User logs in via ADFS
   ↓
UI gets authenticated user identity/JWT
   ↓
UI calls toolboxadmin for authorization list
   ↓
toolboxadmin reads userId from principal/JWT
   ↓
Queries:
   USER_ROLES + ROLE_AUTHORIZATIONS
   ↓
Returns list of authorization codes
   ↓
UI stores auth codes
   ↓
UI filters sidebar, routes, buttons/actions
```

Example:

```text
Has VIEW_ORDER → show order list
Has SAVE_ORDER → show create order
Has RETURN_VEHICLE.PUT → show return button
No CANCEL_ORDER.PUT → hide cancel button
```

---

# 6. What `AuthorizationList.json` Did

It was the master permission catalog.

It defined all valid authorization codes like:

```text
VIEW_ORDER
SAVE_ORDER
MODIFY_ORDER
RETURN_VEHICLE.PUT
SAP_HANDOVER_REPORT
PRICING_MATRIX.POST
```

Then roles were built from these codes.

Purpose:

> Avoid hardcoded random permissions and centralize access definitions.

---

# 7. Request Flow to a Service

Example: Return Vehicle API

```text
User clicks Return Vehicle
   ↓
UI checks RETURN_VEHICLE.PUT
   ↓
If allowed, sends API request
   ↓
Browser includes JWT cookie
   ↓
Gateway routes to mcp-vehicle-order
   ↓
mcp-vehicle-order extracts JWT using cookieTokenResolver
   ↓
JWT validated by Spring Security
   ↓
JwtConverter creates authenticated principal
   ↓
Controller/service executes
```

If no/invalid token:

```text
401 Unauthorized
```

If authenticated but not allowed:

```text
403 Forbidden / UI hides action
```

---

# 8. Key Interview Summary

Say this:

> Authentication was handled by ADFS through OAuth2/OIDC. The frontend redirected users to SSO through `toolboxadfsbridge`. ADFS authenticated the user and returned tokens. The bridge extracted user claims and established a JWT-based session, usually through a secure cookie. Every backend microservice was configured as an OAuth2 resource server and validated the JWT independently.
>
> Authorization was handled by `toolboxadmin`. Users were mapped to roles, roles were mapped to authorization codes, and those codes were seeded from `AuthorizationList.json`. After login, the UI fetched the user’s authorization codes and used them to control sidebar items, routes, and buttons. Backend services validated JWTs and could enforce the same permissions for sensitive APIs.

---

# 9. One-Liner to Memorize

```text
ADFS handled identity, JWT secured API calls, toolboxadmin resolved permissions, and the UI/services enforced role-based access using authorization codes.
```