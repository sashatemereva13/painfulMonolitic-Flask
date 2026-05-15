# GameHub — Understanding the system

10 questions to test your understanding of the data flow and architecture.
Work through them in order: read the code first, then run the app, then try to break things.

---

## How to investigate

You will need three things:

**1. Read the source code**
Start with `models.py` (the schema), then `seed.py` (the data), then `app.py` (the logic).
Many questions are answered entirely by reading carefully.

**2. Run the app and interact with it**
Use the UI at `http://localhost:5000` or send requests with curl or Postman.
Observe what actually happens — don't just reason about it.

```bash
# Example: log an activity for nova (id=1) on Hollow Knight (id=1)
curl -X POST http://localhost:5000/activities \
  -H "Content-Type: application/json" \
  -d '{"user_id": 1, "game_id": 1, "action": "started"}'
```

**3. Query the database directly**
Open `gamehub.db` with a SQLite tool and inspect the actual rows.

```bash
sqlite3 gamehub.db
.tables
SELECT COUNT(*) FROM notifications;
SELECT * FROM notifications WHERE user_id = 1;
```

Or use a GUI: **DB Browser for SQLite** (free, recommended).

---

## Suggested approach

| Phase         | Questions   | What you are doing                           |
| ------------- | ----------- | -------------------------------------------- |
| Read first    | 1, 4, 8, 10 | Understand the code before touching anything |
| Then run it   | 3, 6, 9     | Observe actual behaviour                     |
| Then break it | 2, 5, 7     | Try things, hit walls, reason about why      |

---

## Questions

**1.** When a user logs a new activity, how many database tables are written to?
List them and explain why each one is affected.

**Answer**: coming from

```
    conn.execute(
        "INSERT INTO activities (user_id, game_id, action, created_at) VALUES (?,?,?,?)",
        (user_id, game_id, action, now())
    )
```

the tables that are affected are:
User, Game, Action, Date.

user_id - logs a new user from the list of users to the table Users

game_id - logs a new game from the list of games to the table Games

action - logs a new activity from the list of activities to the table Activities

now() - finds out the now time and logs it to the Date table

---

**2.** You call `DELETE FROM users WHERE id = 3` directly in SQLite.
What happens, and why? What would you need to do instead?

**Answer**:

```
13xsasha@Sashas-MacBook-Pro painfulMonolitic-Flask % sqlite3 gamehub.db
SQLite version 3.51.0 2025-06-12 13:14:41
Enter ".help" for usage hints.
sqlite> DELETE FROM users WHERE id = 3
   ...>
   ...> ;
sqlite>
```

so what happens in this case is that what gets deleted is the user with id=3, BUT what doesn't get deleted are:

- notifications for this user
- notifications about this user
- activities related to this user
- games played by this user
- friends relations

so, basically, the whole environment surrounding the 'non-existing anymore' user stays half-alive like a ghost, forever staying left behind. that could cause some big problems, which actually happened to me before (in a different project), when i tried to log a new user's activities but the id for those activities was already taken by a deleted user. what a mess.

---

**3.** User `nova` changes her username to `nova_2`.
She then checks her friends' notification feeds.
What do they see — the old name or the new one? Why?

**Anwer**:
I changed nova's name from the UI localhost and what i see in alex_g's notifications is this:
nova_2 Your friend did something new — check their activity! 2024-01-15T10:00:00Z Dismiss

so, alex seees nova's new name.

## i assume, there should be a function that updates her name for her frineds' profiles, but i don't see the notification part in the app.route("/users/<int:user_id>", methods=["PUT"]). However, it must have triggered some other function that updated her name everywhere, even for the older status updates. I'll try to find it out.

**4.** Trace the full journey of a `POST /activities` request.
Starting from the HTTP call, list every operation that happens before the response is returned.

**Answer**:
POST /activities triggers function create_activity(), which consists of:

- connection to the db
- selecting and blocking tracking if the user opted out of it
- inserting the activity into the db
- selecting everything related to activities with a particular user id
- assigning actor var to display name
- assignning game var to the title of the game
- in case notifications are enabled, inserting a notification into every friend's table and notifying those users
- closing the db connection
- sending 201 for the job well done

---

**5.** `pixel_queen` opts out of activity tracking.
A teammate adds an `opted_out` boolean column to the `users` table and updates the `POST /activities` API route to check it.
Is the feature fully implemented? What did they miss?

**Answer**:
i've added this line 'opt_out_tracking INTEGER DEFAULT 1' to the users' table creation and the

```
    # block tracking if the user opted out
    if user and user["opt_out_tracking"] == 1:
        conn.close()
        return jsonify({"message": "Tracking disabled"}), 200
```

to the /activities POST route.

The real question is - what did I miss, thinking that was all?
I suppose, the table activities should be linked to this logic, too?
And what i know for sure is that the /view/activities POST should also get these lines:

```
    if user and user["opt_out_tracking"] == 1:
        conn.close()
        return redirect(url_for("view_activities"))

```

---

**6.** How many rows are created in the database when `nova` logs one activity, given the current seed data?
Show your working.

**Answer**:
from this line, "INSERT INTO activities (user_id, game_id, action, created_at) VALUES (?,?,?,?)", there are 4 rows being created.

---

**7.** You need to delete `maya_r`.
In what order must you delete rows across the tables, and why does the order matter?

**Answer**:
so, to delete a user, we must manually remove every row that references that user, but we must start not with deleting the user from db - instead, we shall start with notifications related to them. Notifications reference activities, so activities can't be deleted before notifications. friends reference users twice.

as such, the deletion process becomes lengthy and thorough:

```
    # Step 1 — remove notifications where user is the receiver
    conn.execute("DELETE FROM notifications WHERE user_id = ?", (user_id,))

    # Step 2 — remove notifications where user is the one who triggered them
    conn.execute("DELETE FROM notifications WHERE triggered_by = ?", (user_id,))

    # Step 3 — remove activities
    conn.execute("DELETE FROM activities WHERE user_id = ?", (user_id,))

    # Step 4 — remove from user_games
    conn.execute("DELETE FROM user_games WHERE user_id = ?", (user_id,))

    # Step 5 — remove friendships (both directions)
    conn.execute("DELETE FROM friends WHERE user_id = ? OR friend_id = ?", (user_id, user_id))

    # Step 6 — finally, remove the user
    conn.execute("DELETE FROM users WHERE id = ?", (user_id,))
```

---

**8.** The `notifications` table has a foreign key pointing to `activities`.
What happens if you try to delete an activity that has notifications attached to it?

**Answer**:
as was mentioned during the previous answer, notifications reference activities, so activities can't be deleted before notifications.
what happens if we call for our rebelious nature and try anyways?


---

**9.** A bug is found in the game catalog — wrong genre for one game.
You fix it and restart the app to ship the change.
What else just went down, and for how long?


---

**10.** A teammate says: _"let's just move the notification logic into its own function in `app.py`"_.
Does that solve the problem described in Task 4?
What is the actual architectural issue?
