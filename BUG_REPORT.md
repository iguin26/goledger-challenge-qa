# Bug Report

## BUG-001 - Passwords are stored in plain text and exposed by the profile endpoint

- Component: API
- Endpoint / Page: `POST /auth/register`, `GET /me`
- Severity: Critical
- Description: User passwords are stored in memory as plain text in `api/models/user.go`, and the profile endpoint returns the full user object, including the `password` field. That exposes sensitive credentials to any authenticated client and makes credential compromise trivial.
- Steps to Reproduce:
  1. Register a new account through `POST /auth/register`.
  2. Log in with that account.
  3. Call `GET /me` with the returned JWT.
  4. Observe that the response includes the `password` value.
- Expected Behaviour: Passwords should be hashed before storage, and profile responses should never include password data.
- Actual Behaviour: The application stores the raw password and returns it in the API response.
- Proposed Fix: Hash passwords with a secure algorithm such as bcrypt or Argon2 before persisting them, verify them with a hash comparison during login, and return a sanitized user DTO from `/me` that excludes `password`.

## BUG-002 - JWT secret is hard-coded in the source code

- Component: API
- Endpoint / Page: Authentication flow (`POST /auth/login`, all protected routes)
- Severity: Critical
- Description: The JWT signing key is defined as a constant in `api/config/config.go`. A hard-coded secret can be discovered from the repository and used to forge valid tokens.
- Steps to Reproduce:
  1. Open `api/config/config.go`.
  2. Note that `JWTSecret` is set to a fixed string.
  3. Log in once to obtain a token.
  4. Re-sign a modified JWT with the known secret and send it to a protected endpoint such as `GET /me`.
- Expected Behaviour: The JWT secret should be loaded from environment variables or a secret manager and should not be committed to source control.
- Actual Behaviour: Anyone with access to the codebase can discover the signing secret and mint arbitrary tokens.
- Proposed Fix: Move the JWT secret to `.env` or a secret store, fail fast when the variable is missing in non-development environments, and rotate any tokens issued with the old secret.

## BUG-003 - DELETE /books is publicly accessible

- Component: API
- Endpoint / Page: `DELETE /books`
- Severity: High
- Description: The delete route is registered outside the authentication middleware, so it can be called without a JWT. This allows unauthorized book deletion.
- Steps to Reproduce:
  1. Send a `DELETE /books?title=<title>&author=<author>` request without an `Authorization` header.
  2. Use a title and author that exist in the ledger.
  3. Observe that the request is accepted instead of being rejected with `401 Unauthorized`.
- Expected Behaviour: Deleting a book should require a valid authenticated user.
- Actual Behaviour: The endpoint can be invoked without authentication.
- Proposed Fix: Move `DELETE /books` into the authenticated router group in `api/routes/routes.go` so it is covered by `AuthRequired()`.

## BUG-004 - Book search pagination is broken in the backend

- Component: API
- Endpoint / Page: `GET /books`
- Severity: High
- Description: The pagination logic in `api/handlers/books.go` is incorrect. When no genre filter is provided, the handler leaves `filteredBooks` empty and then paginates against that empty slice, which can return an empty array even when books exist. The offset calculation also starts at `page * limit` instead of `(page - 1) * limit`.
- Steps to Reproduce:
  1. Log in.
  2. Call `GET /books?author=<existing author>&page=1&limit=10` without a `genre` filter.
  3. Observe that the response can be empty even when matching books exist.
  4. Repeat with higher page numbers and compare the returned records.
- Expected Behaviour: Page 1 should return the first set of matching books, and each page should advance by the configured limit.
- Actual Behaviour: The endpoint can return empty results or skip the first page of data.
- Proposed Fix: Use the unfiltered book list when `genre` is empty, compute the offset as `(page - 1) * limit`, and paginate the same collection that is being filtered.

## BUG-005 - Books page pagination controls do not behave correctly

- Component: Web
- Endpoint / Page: Books page
- Severity: Medium
- Description: The UI pagination controls are not aligned with the current page state. The Prev button always resets to page 1 instead of moving back one page, and the Next button is never disabled even when there are no more results.
- Steps to Reproduce:
  1. Open the Books page and run a search that returns multiple pages.
  2. Move to page 2 or page 3.
  3. Click Prev and observe that the page jumps back to 1.
  4. Keep clicking Next until the results are exhausted.
  5. Observe that Next remains enabled.
- Expected Behaviour: Prev should move one page back, and Next should disable when there are no further results.
- Actual Behaviour: Prev always returns to the first page, and Next remains active indefinitely.
- Proposed Fix: Change Prev to decrement the current page by one, and disable Next when the current page does not fill the selected page size:

```tsx
const handleNext = () => {
  const next = page + 1;
  setPage(next);
  handleSearch(next);
};

const handlePrev = () => {
  const prev = Math.max(1, page - 1);
  setPage(prev);
  handleSearch(prev);
};

const hasMoreResults = books.length === limit;
```

```tsx
<button
  className="btn btn-secondary btn-sm"
  onClick={handleNext}
  disabled={loading || !hasMoreResults}
>
  Next →
</button>
```

## BUG-006 - Creating a person is treated as a failure in the frontend

- Component: Web
- Endpoint / Page: Persons page, `POST /persons`
- Severity: Medium
- Description: The frontend helper `createPerson()` only accepts status code `200`, but the backend returns `201 Created` on success. As a result, a successful person creation is reported as an error in the UI.
- Steps to Reproduce:
  1. Open the Persons page.
  2. Fill the CPF and name fields.
  3. Submit the form.
  4. Observe that the request succeeds on the backend but the frontend still shows an error message.
- Expected Behaviour: A successful creation should be accepted as a success state.
- Actual Behaviour: The client throws an error because it does not recognize `201 Created` as success.
- Proposed Fix: Replace the strict `res.status !== 200` check with a success-range check so `201 Created` is accepted as valid:

```ts
export async function createPerson(data: CreatePersonData): Promise<Person> {
  const res = await fetch(`${API_BASE}/persons`, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      Authorization: `Bearer ${getToken()}`,
    },
    body: JSON.stringify(data),
  });

  const body = await res.json();
  if (!res.ok) {
    throw new Error(body.error ?? "Failed to create person");
  }

  return body as Person;
}
```

## BUG-007 - Assigning a tenant to a book uses the read-only chaincode path

- Component: API
- Endpoint / Page: `PUT /books/tenant`
- Severity: High
- Description: The handler for assigning a tenant calls `ccapi.Query(...)` instead of `ccapi.Invoke(...)`. That means a state-changing operation is sent through the read-only path, so the update cannot be persisted correctly.
- Steps to Reproduce:
  1. Log in.
  2. Submit a valid request to `PUT /books/tenant` with an existing book and person ID.
  3. Observe that the handler does not perform a proper ledger update.
- Expected Behaviour: The tenant assignment should mutate the ledger and persist the new association.
- Actual Behaviour: The request is routed through the query path, which is intended for read-only operations.
- Proposed Fix: Use the state-changing CCAPI path so the tenant assignment is written to the ledger:

```go
result, status, err := ccapi.Invoke(
  config.GetCCAPIOrgURL(),
  http.MethodPut,
  "updateBookTenant",
  params,
)
if err != nil {
  c.JSON(http.StatusBadGateway, gin.H{"error": "failed to reach chaincode API"})
  return
}
if status != http.StatusOK {
  c.JSON(status, gin.H{"error": string(result)})
  return
}
```

## BUG-008 - Upstream CCAPI requests have no HTTP timeout

- Component: API
- Endpoint / Page: All CCAPI-backed endpoints
- Severity: Medium
- Description: The HTTP client in `api/ccapi/client.go` is created without a timeout. If the upstream chaincode API hangs or becomes unreachable, the request can block indefinitely.
- Steps to Reproduce:
  1. Point `CCAPI_ORG_URL` to a slow or non-responsive upstream service.
  2. Call any endpoint that uses `ccapi.Query` or `ccapi.Invoke`.
  3. Observe that the request can hang instead of failing fast.
- Expected Behaviour: Requests to the upstream API should time out and return a controlled error.
- Actual Behaviour: The backend waits indefinitely for the HTTP response.
- Proposed Fix: Configure `http.Client{Timeout: ...}` or use a context with deadline for outbound CCAPI requests.

## BUG-009 - Logout does not clear the stored JWT token

- Component: Web
- Endpoint / Page: Navbar logout action
- Severity: High
- Description: The logout flow does not consistently clear the token from localStorage, so a user may remain effectively authenticated after clicking Logout.
- Steps to Reproduce:
  1. Log in to the web app.
  2. Click Logout.
  3. Refresh the page or navigate back to a protected page.
  4. Observe that the session can still persist if the token is not removed.
- Expected Behaviour: Logging out should remove the stored token and fully end the session on the client.
- Actual Behaviour: The token is left behind, allowing the app to keep treating the browser as authenticated.
- Proposed Fix: Call the shared token removal helper in the logout handler and reset any authentication state derived from localStorage.

## BUG-010 - Login compares password length instead of password value

- Component: API
- Endpoint / Page: `POST /auth/login`
- Severity: Critical
- Description: The login handler validates credentials by comparing the length of the submitted password to the stored password instead of comparing the actual values. Different passwords with the same length can authenticate successfully.
- Steps to Reproduce:
  1. Register or use any account with a known password length.
  2. Submit a login request with a different password that has the same number of characters.
  3. Observe that the login can succeed incorrectly.
- Expected Behaviour: The API should validate the exact password value.
- Actual Behaviour: Any password with a matching length is treated as valid.
- Proposed Fix: Compare `req.Password` directly with `user.Password`, or preferably compare a hashed password using a secure password verifier.

## BUG-011 - Creating a book requires authorization, but the frontend did not send the header

- Component: Web
- Endpoint / Page: Books page, `POST /books`
- Severity: High
- Description: The book creation flow depends on an authenticated backend route, but the frontend request omitted the `Authorization` header. That causes the backend to reject the request with `authorization header required`.
- Steps to Reproduce:
  1. Open the Books page.
  2. Fill in the form and submit a new book.
  3. Observe the backend response complaining about the missing authorization header.
- Expected Behaviour: The client should include the current Bearer token when creating a book.
- Actual Behaviour: The request is sent without authentication and is rejected.
- Proposed Fix: Include the JWT in the request headers and keep the existing error handling so the protected route can be reached successfully:

```ts
export async function createBook(data: CreateBookData): Promise<Book> {
  const res = await fetch(`${API_BASE}/books`, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      Authorization: `Bearer ${getToken()}`,
    },
    body: JSON.stringify(data),
  });

  const body = await res.json();
  if (!res.ok) {
    throw new Error(body.error ?? "Failed to create book");
  }

  return body as Book;
}
```
