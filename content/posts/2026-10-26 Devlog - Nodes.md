---
title: "Devlog: The node tree, subtrees and status codes"
date: 2026-10-26
tags: ["java", "javalin", "jpa", "hibernate", "rest", "devlog"]
draft: false
---

With login working, today was about the core of the project: the node tree that a guide is built from. Creating nodes already worked, so the work was updating, moving and deleting them, which all turned out to depend on the same thing: finding a node's whole subtree.

## Starting with a status check

Before writing more code, I went through the project against the "Must" stories in my user stories. Two things came out of it:

- **The pushed version didn't compile.** `Main` used classes I hadn't committed, and `NodeRoutes` pointed at controller methods that didn't exist yet. The lesson is to run `git status` before assuming GitHub has the newest code.
- **`config.properties` was public on GitHub**, with the database password and the JWT secret, even though it was in `.gitignore`. I'll come back to that one.

## Deleting a node and everything under it

A node can have children at any depth, so deleting one has to delete its whole subtree too.

### Finding the subtree

The database only stores the link upwards: every node knows its parent. To walk down the tree, I first turn that around into a map from parent to children:

```java
Map<Long, List<Long>> childrenByParent = new HashMap<>();
for (Node n : nodeDao.findByGuideId(guideId)) {
    if (n.getParent() != null) {
        childrenByParent.computeIfAbsent(n.getParent().getId(), k -> new ArrayList<>()).add(n.getId());
    }
}
```

`computeIfAbsent` creates an empty list the first time a parent shows up and returns the existing list after that, which replaces a `get`, a null check and a `put`.

Then a to-do list walks down from the node being deleted:

```java
Set<Long> ids = new HashSet<>();
Deque<Long> toVisit = new ArrayDeque<>();
toVisit.push(rootId);
while (!toVisit.isEmpty()) {
    Long current = toVisit.pop();
    if (ids.add(current)) {
        toVisit.addAll(childrenByParent.getOrDefault(current, List.of()));
    }
}
```

- `findByGuideId` gets all of the guide's nodes in **one query**, however deep the tree is.
- `getOrDefault(..., List.of())` returns an empty list for nodes without children, since `addAll(null)` would throw.
- `Set.add` returns `false` if the id was already there, so the same line both collects the id and stops an infinite loop if the data ever contains one.

### Deque, push and pop

`Deque` is a double-ended queue. `push` and `pop` work on the front, while `addAll` and `offer` add at the back. Mixing `addAll` with `pop` makes it behave like a queue, so the tree is visited level by level: the node, then all its children, then the grandchildren. That's breadth-first search. The order doesn't matter here, since I only need the set of ids, but `offer` and `poll` would make the queue behaviour clearer to read.

### The delete itself

```java
public NodeDTO deleteNode(long id, AuthUser authUser) {
    Node node = findNodeOrThrow(id);
    Guide guide = findGuideOrThrow(node.getGuide().getId());
    securityService.verifyAccess(guide.getAuthor(), authUser);
    nodeDao.deleteAll(findSubtreeIds(id, guide.getId()));
    return NodeMapper.toAuthorDTO(node);
}
```

- **`node.getGuide()` is a lazy proxy**, and the `EntityManager` is already closed after `findById`. Reading the proxy's id is fine, but `node.getGuide().getAuthor()` would throw a `LazyInitializationException`, so the guide is loaded again.
- **One `DELETE ... WHERE id IN (...)`** removes the node and its children together. Postgres checks the `parent_id` foreign key after the whole statement, so the order inside the list doesn't matter.
- **Reveals disappear by themselves**, because `Reveal` has `@OnDelete(action = OnDeleteAction.CASCADE)` on its node.

## Updating and moving nodes

`updateNode` takes a `NodeRequest`, and every field that's `null` is left unchanged, so the client only sends what it wants to change.

Changing `parentId` moves a node, and its children come along automatically, since they still point at it. Two moves have to be rejected with a 400:

- **To a parent in another guide.**
- **Under itself or one of its own descendants.** If node 5 were moved under its own grandchild 12, the nodes would only hang from each other and disappear from the guide. The check reuses the subtree: if the new parent is in the node's subtree, it's a loop.

One limitation: since `null` means "unchanged", a node can't be moved back to the top level. That's fine for the MVP.

## Routes, path parameters and status codes

- **`{id}` in `path("nodes/{id}", ...)`** catches whatever comes after `/nodes/`, and the controller reads it with `ctx.pathParamAsClass("id", Long.class)`. The name has to match exactly. A value like `abc` gives a 400 by itself.
- **`PUT` and `DELETE` share the same path**, and the HTTP method decides which handler runs.
- **Shallow nesting.** Creating a node needs the guide (`POST /guides/{guideId}/nodes`), but updating and deleting don't, because a node's id is unique and the node knows its guide. So those are just `/nodes/{id}`.
- **Status codes:** 201 for create, 200 for read, update and delete when a body is returned, and 204 when there's no body. Javalin uses 200 by default, so only create needs `ctx.status(201)`.
- **The `OPTIONS` routes in the startup log** aren't mine. `enableHttpAllowedMethodsOnRoutes()` adds one per path, so browsers can ask which methods are allowed before a cross-origin request (a CORS preflight). They have no role, since a preflight never carries a token.

## HTTP tests

I wrote `Nodes.http` with a small tree (Act 1 → Limgrave → Church of Elleh, plus Act 2) and a test for every success and error case. Two things got in the way:

- **"Unsubstituted variable 'guideId'".** IntelliJ's variables are only set when the request that sets them runs. Running one request on its own skips the setup above it, so **Run All Requests in File** is the way to run the file.
- **`Auth.http` didn't log in author and otheruser**, so their tokens were always empty. I added both logins and made `Auth.http` create the shared setup guide.

## Next steps

- [ ] Reveal endpoints: reveal a node and its subtree, hide it again
- [ ] Make sure guests and logged-in users get their own reveals
- [ ] A `Game` entity, so guides belong to a game
- [ ] A published/draft status for guides
- [ ] Stop tracking `config.properties` and change the exposed secrets
- [ ] JUnit tests for the subtree logic
