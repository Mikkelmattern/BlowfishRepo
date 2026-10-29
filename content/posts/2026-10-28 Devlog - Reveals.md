---
title: "Devlog: Reveals, a shared session bug and the RAWG API"
date: 2026-10-28
tags: ["java", "javalin", "jpa", "jwt", "api", "security", "devlog"]
draft: false
---

Today the project got its main feature: revealing sections of a guide. It also got games from the RAWG API, and a clean-up of the secrets I'd pushed to GitHub. The most useful lesson came from a test that failed for a reason I didn't expect.

## The reveal endpoints

| Request | What it does |
|---|---|
| `POST /nodes/{id}/reveal` | Reveal a node and everything under it |
| `DELETE /nodes/{id}/reveal` | Hide a node and everything under it again |
| `DELETE /guides/{id}/reveals` | Reset all reveals for a guide |

All three are open to anyone, since guests should be able to use a guide without an account.

- **The cascade reuses the subtree code** from the node delete, so revealing Act 1 reveals everything under it at any depth. Revealing Limgrave only reveals Limgrave and its children. The parent and siblings stay hidden.
- **Logged-in users' reveals are saved** as `Reveal` rows, one per user and node.
- **Already revealed nodes are skipped.** `Reveal` has a unique constraint on `(user_id, node_id)`, so revealing a parent after one of its children would otherwise break it. The service loads the ids that are already revealed and only saves the new ones.
- **Guests' reveals live in the session.** The controller reads a `Set<Long>` from `ctx.sessionAttribute("reveals")`, the service adds to it or removes from it, and the controller stores it again. One set covers every guide, since node ids are unique across guides.
- **The service never touches the `Context`.** It gets the guest's set as a parameter, so it stays testable without Javalin.

`GET /guides/{id}` already used the revealed ids: a node that isn't revealed is sent with `"revealed": false` and `"content": null`, so the content never reaches the client before the reader asks for it.

## The bug: everyone was a guest

My `Reveals.http` test failed with *"Author should not see otheruser's reveals"*. Otheruser had revealed Act 1, and the author, who had revealed nothing, could see it.

The cause was in `authenticate`. It started like this:

```java
if (isOpenRoute(ctx) || ctx.method() == HandlerType.OPTIONS) {
    return;
}
```

On an open route, it returned **before reading the token**. Both the reveal endpoints and `GET /guides/{id}` are open, so `authUser` was always `null` there, even with a valid `Authorization` header, and every request was treated as a guest. On top of that, IntelliJ's HTTP client keeps one `JSESSIONID` cookie for all requests, so all the "guests" shared the same session. Otheruser's reveal never went into the database, but into a session everyone used.

The first test only passed by luck, because the session made it look as if the reveals had been saved.

The fix: open routes still don't need a token, but if one is sent, it's read.

```java
if (isOpenRoute(ctx)) {
    String header = ctx.header("Authorization");
    if (header != null && header.startsWith("Bearer ")) {
        try {
            ctx.attribute("user", tokenService.verify(header.substring("Bearer ".length())));
        } catch (ApiException e) {
            // An invalid or expired token on an open route just means the request is a guest's
        }
    }
    return;
}
```

After the fix, I had to restart the server to clear the old session, since it still held the guest reveals from earlier runs.

The lesson: a test that checks that one user's data doesn't leak to another caught a bug that every "happy path" test missed.

## Games from RAWG

Guides now belong to a game. Instead of typing game data in by hand, an admin searches RAWG and imports the game.

| Request | Role |
|---|---|
| `GET /games/search?query=...` | Admin |
| `POST /games/import/{rawgId}` | Admin |
| `GET /games`, `GET /games/{id}` | Anyone |
| `GET /games/{id}/guides` | Anyone |

- **`RawgClient`** uses Java's `HttpClient` to call the API and Jackson's `ObjectMapper` to turn the JSON into records (`RawgSearchResultDTO`, `RawgGameDTO`). The mapper needs `JavaTimeModule` for the release date.
- **RAWG errors become my own errors.** A game RAWG doesn't know gives 404, and RAWG being down gives 502, so a client never sees a raw `IOException`.
- **`rawgId` is unique on `Game`**, so the same game can't be imported twice. A second import gives 409 Conflict.
- **The API key** is read from `config.properties`, like the other secrets.

`Auth.http` now searches for and imports four games and creates the setup guide for Elden Ring. The imports accept both 201 and 409, so the file still works if the server hasn't been restarted.

## Cleaning up the secrets

`config.properties` had been on GitHub since my first commits, with the database password and the JWT secret in it, even though it was in `.gitignore`.

- **`.gitignore` only stops Git from *starting* to track a file.** A file that's already committed stays tracked. `git rm --cached` removes it from Git but keeps the local file, and from then on `.gitignore` takes over.
- **The old values are still in the history.** Removing a file from the latest commit doesn't remove it from older ones. Rewriting history with `git filter-repo` and a force push is possible, but the real fix is to change the secrets, so the old ones are worthless.
- **New database password:** `ALTER USER build_guide_user WITH PASSWORD '...';`, then the same value in `config.properties`, and a restart so the connection pool drops its old connections.
- **New `SECRET_KEY`:** since it isn't stored in the database, changing it in the file is enough. Every existing token stops working, which is the point.
- **The RAWG key was never pushed**, because the file was untracked before I added it.

## Smaller things I learned

- **A line in `.gitignore` without wildcards matches the exact name only**, so `config.properties` doesn't ignore `config.properties.example`. That one is committed with empty values, so others know which keys to fill in.
- **`docker compose config` prints every variable it resolves**, including secrets. Useful for checking, but don't paste the output anywhere.
- **Reading the debug log beats guessing.** Going through every `Request:` and `Response:` line in Javalin's dev log showed exactly which test failed and why.

## Next steps

- [ ] Test deleting a guide that has a full node tree and saved reveals
- [ ] Published/draft status for guides, so readers only see finished work
- [ ] A way for authors to find their own drafts
- [ ] JUnit tests for the subtree logic
- [ ] A README
