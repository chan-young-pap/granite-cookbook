# 03 — Firebase RTDB rules: room-scoped access with an immutable `authorId`

A pattern for small-group mini-apps where users join an invite-coded room, write records (mood check-ins, time capsules, etc.), and need to read each other's data — but **only** within the room.

This is the rules schema I'm using in production for a daily mood / time-capsule app with up to 8 members per room. It enforces:

- Only authenticated users can touch anything
- Reads/writes inside `/rooms/{roomId}` are limited to room members
- A record's author can never be silently rewritten
- An existing member can add new members (room joining works); a non-member cannot
- Invite codes are world-readable to authenticated users (needed to resolve `code → roomId` on join), but only existing members can mint them

## Schema

```jsonc
{
  "rules": {
    "rooms": {
      "$roomId": {
        ".read":  "auth != null && data.child('users').child(auth.uid).exists()",
        ".write": "auth != null && (!data.exists() || data.child('users').child(auth.uid).exists() || newData.child('users').child(auth.uid).val() === true)",

        "users": {
          "$uid": {
            ".write":    "auth != null && ($uid === auth.uid || data.parent().child(auth.uid).exists())",
            ".validate": "newData.val() === true || newData.val() === null"
          }
        },

        "records": {
          "$date": {
            "$uid": {
              ".write":    "auth != null && auth.uid === $uid && root.child('rooms').child($roomId).child('users').child(auth.uid).exists()",
              ".validate": "newData.hasChildren(['mood', 'timestamp'])"
            }
          }
        },

        "capsules": {
          "$capsuleId": {
            ".write": "auth != null && root.child('rooms').child($roomId).child('users').child(auth.uid).exists()",
            ".validate": "newData.hasChildren(['id','authorId','message','openDate','duration','opened','createdAt']) && (!data.exists() ? newData.child('authorId').val() === auth.uid : newData.child('authorId').val() === data.child('authorId').val())"
          }
        }
      }
    },

    "inviteCodes": {
      "$code": {
        ".read":     "auth != null",
        ".write":    "auth != null && (!data.exists() || root.child('rooms').child(newData.child('roomId').val()).child('users').child(auth.uid).exists())",
        ".validate": "newData.hasChildren(['roomId','createdAt'])"
      }
    },

    "users": {
      "$uid": {
        ".read":  "auth != null",
        ".write": "auth != null && auth.uid === $uid"
      }
    }
  }
}
```

## How the room-write rule reads

```
auth != null
&& (
     !data.exists()                                       // anyone signed-in can create a brand-new room
     || data.child('users').child(auth.uid).exists()      // existing member can edit anything in the room
     || newData.child('users').child(auth.uid).val() === true   // existing-but-joining-this-write
   )
```

That third branch is the join handshake: a non-member can flip exactly one bit, `/rooms/$rid/users/$me = true`, **in the same write that admits them**. Any other path inside the room is closed to them until they're a member.

## How the capsule `.validate` enforces immutability

```
(!data.exists()
  ? newData.child('authorId').val() === auth.uid       // on create: author = caller
  : newData.child('authorId').val() === data.child('authorId').val())  // on update: author cannot change
```

Combined with the create rule (`auth.uid === $uid` for records, room-membership for capsules), this means:

- The author field is stamped server-side on creation
- Subsequent edits are allowed (e.g. the `opened: true` flag flips when someone reads the capsule), but the `authorId` is locked

This is the pattern I'd reach for any time you have a "post / message / pledge" entity where authorship is part of the trust model and you don't want any path that could rewrite it.

## Common mistakes I made before settling here

- **Putting `.indexOn` in the wrong scope.** RTDB silently runs unindexed queries — slow, no warning. Add `.indexOn` per query path you actually use.
- **Forgetting that `.read`/`.write` cascades.** A `.read: false` on a parent doesn't get overridden by `.read: true` on a child. Plan top-down.
- **Treating `.validate` as a security boundary.** It isn't — `.validate` only runs when `.write` passes. If you let writes through, `.validate` won't save you.
- **No `.indexOn` for `createdAt`.** Time-ordered lists were degrading silently until I added it.

## Deploy gotcha

Editing `database.rules.json` locally does **nothing** until you publish. Either:

```bash
firebase deploy --only database
```

or paste the file into the Firebase console's Rules tab and hit **Publish**. I lost an embarrassing amount of time to assuming the local save was enough.

## Related

- A future recipe will cover the anonymous-auth onboarding flow these rules pair with (`signInAnonymously` → write `/users/$uid/profile` → join via invite code).
