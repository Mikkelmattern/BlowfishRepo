---
title: "Devlog: Drafts, publishing and getting ready to hand in"
date: 2026-10-29
tags: ["java", "javalin", "jpa", "rest", "testing", "devlog"]
draft: false
---

The last "Must" story was W7: authors should be able to publish and unpublish guides, so readers only see finished work. After that, the day was about tests and documentation for the hand-in.

## Deleting a guide with a full tree

First I checked something I'd been unsure about: whether deleting a guide works when it has nested nodes and saved reveals. It does. `cascade = ALL` on `Guide.nodes` removes the nodes, and `@OnDelete(CASCADE)` on `Reveal` lets the database remove the reveals.

Getting there took two wrong turns, both in the test file rather than the code:

- **404 on a guide I'd just made.** IntelliJ's HTTP client keeps global variables between runs, so `guideId` still held an id from before a restart. Since `hbm2ddl.auto` is `create`, the database is empty after every restart.
- **403 on the delete.** I'd sent it with the wrong user's token. The guide belongs to the author, so a 403 for someone else was the API doing exactly what it should.

## Drafts

A new guide now starts with `published = false`. Lombok generates `isPublished()`, not `getPublished()`, for a `boolean`.

### Who can see a draft

Only the author and admins. Everyone else gets **404, not 403**, so they can't tell that a draft with that id exists:

```java
if (!guide.isPublished() && !securityService.hasAccess(guide.getAuthor(), authUser)) {
    throw new ApiException(404, "Could not find the guide");
}
```

The check goes at the top of `getGuideById`, before nodes and reveals are loaded. My first versions had the reveal logic twice and two bugs:

- **I compared the logged-in user with themselves.** `hasAccess(userDao.findByID(authUser.id()), authUser)` asks "is this user themself?", which is always true. It has to be compared with `guide.getAuthor()`.
- **Guests crashed.** On an open route, `authUser` is `null` for a guest, so `authUser.id()` threw a `NullPointerException`, which became a 500.

### A boolean version of verifyAccess

`verifyAccess` returns `void` and throws a 403, so it can't be used in an `if`. I added `hasAccess`, which returns a `boolean`, and made `verifyAccess` call it, so the rule only lives in one place:

```java
public boolean hasAccess(User user, AuthUser authUser) {
    if (authUser == null) {
        return false;
    }
    return verifyAdminAccess(authUser) || verifyPersonalAccess(user, authUser);
}
```

The `null` check was never needed before, because `verifyAccess` was only used on routes that require a login, where `authorize` stops guests with a 401 before the service runs.

This isn't overloading. Overloading means the same name with different parameters, and Java can't tell two methods apart by return type alone. A common convention is `hasX`/`canX` for a boolean and `verifyX`/`requireX` for a method that throws.

### Lists only show published guides

`getAll` and `getGuidesByGame` filter in the query with `WHERE g.published = true`. `find` doesn't, because publishing, editing, deleting and adding nodes all need to find drafts. The DAO just fetches data, and the service decides who may see it.

That left authors with no way to find their own drafts, so I added `GET /guides/mine`. It uses `findByAuthorId` without a filter, and the id comes from the token, so a user can only ever get their own guides. There's no id in the URL that someone could change.

## Publishing

| Request | What it does |
|---|---|
| `POST /guides/{id}/publish` | Publish the guide |
| `DELETE /guides/{id}/publish` | Make it a draft again |

Both call one service method, `setPublished(id, boolean, authUser)`. The route decides the value, not the client.

I considered a toggle (`setPublished(!guide.isPublished())`), but then the same request does two different things depending on the current state, and a retried or double-clicked request flips it back. With explicit routes, calling publish twice is harmless. That's what *idempotent* means, and it's what's expected of `PUT` and `DELETE`.

It also follows the same pattern as `/nodes/{id}/reveal`: `POST` turns something on and `DELETE` turns it off.

### HTTP methods are a convention

Javalin uses the method only to pick a handler. A `DELETE` route could create a guide if I wrote it that way. The reason not to is that other software relies on the convention: `GET` must never change anything, since browsers and link previews fetch `GET` URLs on their own, and `PUT` and `DELETE` may be retried automatically because they're expected to be idempotent.

## Three bugs from the review

Reading through the pushed code found three bugs that would have broken the feature:

- **The wrong path parameter.** `publish` and `unpublish` read `pathParamAsClass("gameId", ...)`, but the route is `/guides/{id}/publish`. Copied from the method above it.
- **The missing `null` check** in `hasAccess`, so a guest opening a draft got a 500.
- **Route order.** `/guides/mine` was registered after `/guides/{id}`. Javalin uses the first route that matches, and `{id}` matches any text, including `mine`. Javalin has no way to say "numbers only" in a path, so fixed paths have to come before placeholders.

All three would have shown up immediately with a test for each, which is what `Publish.http` now has: drafts hidden from guests and other users, visible to the author and admins, left out of the lists, `/guides/mine`, publishing and unpublishing, and an admin unpublishing someone else's guide.

Since guides now start as drafts, `Auth.http` and `Guides.http` publish their guides right after creating them. Otherwise every other test reading them as a guest would get 404.

## Unit tests

`TreeUtilTest` tests the subtree logic that reveal, hide, move and delete all depend on: the whole subtree from a top-level node, only the children from a node in the middle, leaves, a chain 100 levels deep, a node with 50 children, the order of the list, an empty list, and a loop in the data. It only uses `new Node()` and setters, so it needs no database.

The test file has to sit in `test/java/mikkelmattern/utils`, matching its `package` line. In IntelliJ, Alt+Enter on the package line moves it there.

## The README

The README follows the school's structure: project overview, architecture, data model, API and user stories. The architecture diagram and the ERD were written in Mermaid, but the viewer I used showed them as code, so I rendered them as PNGs in a `docs` folder, which look the same everywhere.

All 19 "Must" stories are done, plus R5, R7, A7, M2 and M3. The rest are mostly frontend features, password reset, spoiler titles and the map phase.

## Next steps

- [ ] Add the loop guard to `TreeUtil` (`if (result.add(id))`), so the loop test passes
- [ ] Stop `notFound()` from replacing my own 404 messages with "Not found"
- [ ] Stop tracking `.idea/workspace.xml`
- [ ] Decide whether to keep `savedGuides`, which has no endpoints yet
- [ ] Check that `mvn test` runs `TreeUtilTest` from the terminal
- [ ] Add a Build Guide page to the portfolio
