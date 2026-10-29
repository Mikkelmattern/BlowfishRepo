---
title: "Devlog: Users, security and wiring the API"
date: 2026-10-06
tags: ["java", "javalin", "jpa", "hibernate", "security", "devlog"]
draft: false
---

Today was mostly about the `User` side of the guide project: getting the entity right, preparing it for login, and connecting the security pieces to the rest of the app. A lot of small bugs turned out to have one thing in common: the code compiled fine and quietly did the wrong thing.

## The User entity

### Table name

`user` is a reserved word in PostgreSQL, so the entity got `@Table(name = "users")`. JPQL still uses the class name (`SELECT u FROM User u`). Only raw SQL uses the table name.

### IDs

Switched from the default `@GeneratedValue` (which uses a sequence on PostgreSQL) to `GenerationType.IDENTITY`, so it matches the other entities. Since `hbm2ddl.auto=update` doesn't change existing columns, the tables had to be dropped and recreated.

### Passwords

- The password is hashed with BCrypt in the constructor, so a user can never be created with a plain-text password.
- `@Builder` is placed on the constructor, not the class. With `@Builder` on the class, Lombok builds through an all-args constructor and skips the hashing completely.
- The field is now `hashedPassword`, with `@Setter(AccessLevel.NONE)` and `@Getter(AccessLevel.NONE)`. The generated setter would have stored plain text, and nothing outside the entity needs to read the hash.
- `setPassword` hashes too. The first version assigned the hash to the parameter instead of the field, so it did nothing at all.
- `@ToString.Exclude` on the hash, so it never ends up in logs.

### verifyPassword

The first version compared the stored hash to the typed password with `equals`, so every login would have failed. It now uses `BCrypt.checkpw(pw, hashedPassword)`, which re-hashes the input with the stored salt. Because the check lives in the entity, the hash never has to leave the object.

### Roles

Roles are a `Role` enum stored with `@ElementCollection`, `@CollectionTable(name = "user_roles")` and `@Enumerated(EnumType.STRING)`. `@JoinTable` is only for relationships between two entities, so it doesn't apply to an enum.

The roles are loaded `EAGER`, since a user only has a few and they're needed almost every time.

`getRolesAsStrings()` first returned `EnumSet.allOf(Role.class)`, which is every role that exists rather than the user's own. That would have made every user an admin. It now streams the user's `roles` field and maps with `Role::name`. I used `name()` rather than `toString()`, because `toString` can be overridden.

### Collections

The guide collections are now `Set`s, initialized with `new HashSet<>()`. The constructor only overwrites them when a value is actually passed. One of them had a stray `{ }` after `new HashSet<>()`, which made it an anonymous subclass instead of a normal set.

## DAOs

- **A shared parent DAO.** The DAOs now extend a parent class. The child constructor has to call `super(emf)` as its first line, because the parent has no no-args constructor. The duplicate `emf` field in the child was removed.
- **`em.find` only searches by primary key.** `findUserByName` uses a JPQL query with `getResultStream().findFirst().orElse(null)`, so an unknown name gives `null` instead of an exception.
- **Everything that needs the `EntityManager` goes inside the lambda.** `executeQuery(em -> em.createQuery(...)).getResultList()` ran the query after the `EntityManager` was already closed. `.getResultList()` now sits inside the lambda.
- **`findUserByName` is public and declared in the interface.** The service calls it, so it can't be private, and the service depends on the interface rather than the implementation.

## Security

- **A separate `SecurityService`** for login and register, kept apart from `UserService`, which handles the user features.
- **The login flow:** find the user by username, then call `user.verifyPassword(password)`. If the user doesn't exist or the password is wrong, it throws `ApiException(401, "Invalid username or password")`. Both failures get the same message, so the endpoint doesn't reveal which usernames exist.
- **`login` returns the `User`, not a boolean,** since the user's name and roles are needed for the token.
- **`Role` implements Javalin's `RouteRole`.** `enableRouteOverview` and route roles only accept `RouteRole`. It's part of Javalin itself, so no extra dependency was needed.
- **`ISecurityUser` lives in the `security` package**, while `User` stays in `entities` and implements it.
- **`getInstance` belongs in `SecurityController`, not in the `ISecurityController` interface.** A static method in an interface must have a body, so it can't be left for the class to implement.

## Wiring the app

- **`ApplicationConfig` collects config steps** and applies them inside `Javalin.create(...)`. Each feature is switched on in a chain: `.cors()`, `.security()`, `.apiExceptions()`, `.exceptions()`, `.notFound()`, `.route(...)`, `.start(7070)`.
- **Everything is under `/api`** because of `contextPath`, so the `.http` files need `@baseUrl = http://localhost:7070/api`.
- **`beforeMatched` runs `authenticate` and `authorize`** only for requests that match a route. That means unknown paths get a 404 instead of a 401, and `authorize` can read the route's roles.
- **Controllers use instance methods** and get their service through the constructor. The `EntityManagerFactory` is created once in `HibernateConfig`, fetched in `Main`, and passed to the DAOs.

## Things I learned along the way

- **`x.equals(y)` throws if `x` is `null`.** Put the constant first (`"ADMIN".equals(role)`), or use `Objects.equals`.
- **Sets don't throw on duplicates.** `add` returns `false`, and `remove(obj)` doesn't need a loop. `removeIf` works when matching on a property.
- **`Role.valueOf(...)` is case-sensitive** and throws on unknown names, so input from requests should be normalized or handled.
- **A missing return type makes Java read a method as a constructor,** which is why IntelliJ complained that `static` wasn't allowed.

## Next steps

- [ ] Finish `authenticate` and `authorize` in `SecurityController`
- [ ] Add JWT creation on login, with the user's name, roles and an expiry
- [ ] Load the token secret from an environment variable, not from code
- [ ] Add the `/auth/login` and `/auth/register` routes
- [ ] Create `src/main/resources/public` so the static files config doesn't fail on startup
- [ ] Write `.http` tests for login and protected routes