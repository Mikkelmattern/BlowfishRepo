---
title: "Devlog: Routes, tokens and authorization"
date: 2026-10-07
tags: ["java", "javalin", "jwt", "security", "devlog"]
draft: false
---

Yesterday was about getting the `User` entity ready for security. Today I moved on to what happens on each request: routes with roles, checking who someone is, and deciding what they may do. Most of the time went into understanding how JWTs work and how the pieces get wired together.

## User routes

`UserRoutes.getRoutes()` returns an `EndpointGroup` that `ApplicationConfig` registers with `.route(...)`. Each route has a role:

| Request | Handler | Role |
|---|---|---|
| `GET /api/users` | `getAll` | ADMIN |
| `POST /api/users` | `create` | USER |
| `GET /api/users/{id}` | `getById` | ANYONE |
| `PUT /api/users/{id}` | `update` | USER |
| `DELETE /api/users/{id}` | `delete` | ADMIN |

The first version had `path("guides")` with `userController` handlers, left over from copying the guide routes. The path is now `users`, plural, to match `guides`.

Things I still need to decide:
- New users should sign up through `/auth/register` with `Role.ANYONE`. As it is, `POST /users` requires being logged in.
- `Role.USER` on `update` means *any* logged-in user. Checking that a user only edits their own account belongs in the service.

## Authentication and authorization

Both run through `beforeMatched`, so they only run for requests that match a route, and they can read that route's roles.

### authenticate: who are you?

1. Skip open routes (`Role.ANYONE`) and CORS preflight requests (`OPTIONS`).
2. Read `Authorization: Bearer <token>` from the header.
3. Verify the token's signature and expiry, and read the username and roles from it.
4. Store the result in `ctx.attribute("user")` for `authorize` and the controllers.

If anything fails, the answer is **401 Unauthorized**.

### authorize: are you allowed?

1. Skip open routes.
2. Get the user from `ctx.attribute("user")`.
3. Compare the user's roles with `ctx.routeRoles()`.
4. If they have none of the route's roles, the answer is **403 Forbidden**.

### Checking whether a route is open

```java
private boolean isOpenRoute(Context ctx) {
    return ctx.routeRoles().contains(Role.ANYONE);
}
```

This only works in `beforeMatched`. In a plain `before`, no route has been matched yet, so the set is empty. A route with no role at all counts as closed, which is a safe default.

## Tokens (JWT)

- **A separate `TokenService`.** `SecurityService` handles users (login, register), while `TokenService` handles tokens and needs no database. The token code can then be reused as is.
- **Nimbus JOSE + JWT with HS256.** `createToken` builds the claims (subject, roles, issue time and a 30-minute expiry), signs them with `MACSigner` and calls `serialize()` to get the `header.payload.signature` string.
- **`verify`** parses the token, checks the signature with `MACVerifier`, checks the expiry, and returns the username and roles. Any failure becomes a 401.
- **`AuthUser(username, roles)` instead of `UserDTO`.** The token has no id or email, so it can't fill a `UserDTO`. A small record shaped like what the token holds also keeps the `security` package independent of the app's DTOs.

### The secret

- It's a random string of at least 32 characters, generated with `SecureRandom`, not something I made up.
- It's read from the `SECRET_KEY` environment variable: an IntelliJ run configuration locally, and a `.env` file for Docker on the droplet. It's never committed to Git.
- Only the server knows it. Clients get tokens signed with it, never the secret itself. Changing the payload breaks the signature, so roles in a token can't be forged.

## Wiring the security controller

- **Javalin handlers take exactly one parameter, `Context`.** I tried `authenticate(Context ctx, String secret)`, and then `beforeMatched(securityController::authenticate)` couldn't resolve the method. The token comes from `ctx.header(...)`, and the `TokenService` comes from a field on the controller.
- **A singleton with a parameter is an awkward mix.** `getInstance(tokenService)` ignores its argument after the first call. The two consistent choices are a public constructor called once in `Main` and passed into `ApplicationConfig`, or a real no-argument singleton that reads `SECRET_KEY` itself. I prefer the first, which is the one I have used, since the rest of the app already passes dependencies through constructors.

## Smaller things I learned

- **JetBrains `@NotNull` on parameters** is a hint for IntelliJ, not a runtime check. Jakarta's `@NotNull` is a different annotation that's used for validation.
- **The exception type you throw decides the response.** An `ApiException` gets its own status, and anything else becomes a 500. When converting an exception, pass the original as the `cause`, so the stack trace keeps the real reason. Don't wrap a `NullPointerException`, though, because that's a bug to fix.
- **Mapping stays in the service.** `UserService.getAllUsersInDTO()` maps entities to DTOs, and the controller only calls it.
- **`{"status": 200, "users": [...]}` works, but the status is already in the HTTP response.** `ctx.json(list)` is simpler. A wrapper object is mainly useful if paging is coming later.
- **`@Builder` on the `User` constructor hashes on `.build()`.** `.password("...")` only stores the value, and the constructor hashes it. Leaving out the password crashes, and roles have to be added after `build()` with `addRole(Role.USER)`.

## Next steps

- [ ] Finish `TokenService` and add the `nimbus-jose-jwt` dependency
- [ ] Give `SecurityController` its `TokenService` through the constructor, and pass the controller into `ApplicationConfig`
- [ ] Implement `login` (returns a token) and `register` (default role `USER`) with `/auth` routes on `Role.ANYONE`
- [ ] Decide whether `ADMIN` should automatically pass `USER` routes
- [ ] Add ownership checks for updating and deleting users
- [ ] Write `.http` tests: login, then requests with `Authorization: Bearer {{token}}`, checking for 401 and 403