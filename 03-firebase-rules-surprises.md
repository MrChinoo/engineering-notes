# Three Firebase rules that behave differently than they read

## The most specific match doesn't win

```text
match /config/{id} { allow read: if true; }
match /config/private { allow read: if isAdmin(); }
```

This looks like a public collection with one protected document. Firestore allows a request if **any** matching rule allows it, so `private` is public.

**Fix:** exclude the document in the wildcard: `allow read: if id != 'private';`.

## A condition on the document ID breaks list queries

```text
match /items/{code} {
  allow read: if isAdmin() && code.matches('^[a-z0-9]+$');
}
```

A `get` works. A collection query fails with `Missing or insufficient permissions`, even for an admin, because rules evaluate queries as constraints and this one can't be proven for every document.

**Fix:** keep `list` separate: `allow list: if isAdmin();`.

I missed this because my tests only checked that a stranger couldn't list. Now they also check that an admin can.

## `allow write` includes delete in Storage

```text
allow write: if request.resource.size < 15 * 1024 * 1024;
```

On a delete, `request.resource` is null, so the condition fails and every delete is denied. Orphaned files pile up with no visible error.

**Fix:** separate them:

```text
allow create, update: if request.resource.size < 15 * 1024 * 1024;
allow delete: if isAdmin();
```

## A related trap

A rule like `allow read: if request.auth != null` passes an anonymous test and still lets anyone with a Google account read the data. Test deployed rules both ways with [firestore-rules-probe](https://github.com/MrChinoo/firestore-rules-probe).
