# Examples

Real-world code examples showing how the Torvalds doctrine behaves in practice. The pattern is always the same: design the data first, keep the patch small, and prove the result.

This file should not sound like a polite design review from a corporate slide deck. If the code is garbage, say it's garbage. If the API is brain-damaged, say so. If the patch is random churn, reject it.

---

## 1. Data First

### Example 1: Hidden Assumptions

**User Request:** "Add a feature to export user data"

**❌ What LLMs Do (Guess and Ship)

```python
def export_users(format='json'):
    users = User.query.all()

    if format == 'json':
        with open('users.json', 'w') as f:
            json.dump([u.to_dict() for u in users], f)
    elif format == 'csv':
        with open('users.csv', 'w') as f:
            writer = csv.DictWriter(f, fieldnames=['id', 'email', 'name'])
            writer.writeheader()
            writer.writerows([u.to_dict() for u in users])
```

**Problems:**
- Assumed all users should be exported
- Assumed file output was the right target
- Assumed the field list without checking the model

**✅ What Should Happen**

```
Before coding, I need to know:

1. Export all users or a filtered subset?
2. File download, API response, or background job?
3. Which fields are allowed to leave the system?
4. How much data are we talking about?

Simplest safe version: define the payload shape first, then implement the export path.
```

### Example 2: Better Data Layout

**User Request:** "Delete a node from a singly linked list"

**❌ What LLMs Do (Special-Case Branches)**

```c
void delete(struct node **head, struct node *entry) {
    struct node *prev = NULL;
    struct node *cur = *head;

    while (cur && cur != entry) {
        prev = cur;
        cur = cur->next;
    }

    if (!cur)
        return;

    if (prev)
        prev->next = cur->next;
    else
        *head = cur->next;
}
```

**✅ What Should Happen**

```c
void delete(struct node **head, struct node *entry) {
    struct node **p = head;

    while (*p != entry)
        p = &(*p)->next;
    *p = entry->next;
}
```

**Why this is better:**
- No head special case
- Same operation for every node
- Less branching, less code, less room for nonsense

---

## 2. Simplicity First

### Example: Over-Engineered "Flexibility"

**User Request:** "Save user preferences to the database"

**❌ What LLMs Do (Invent a Framework)**

```python
class PreferenceManager:
    def __init__(self, db, cache=None, validator=None):
        self.db = db
        self.cache = cache or InMemoryCache()
        self.validator = validator or DefaultValidator()

    def save(self, user_id, prefs, merge=True, notify=False):
        if self.validator:
            self.validator.validate(prefs)
        if merge:
            existing = self.get(user_id)
            existing.update(prefs)
            prefs = existing
        self.db.execute(
            'UPDATE users SET preferences = ? WHERE id = ?',
            (json.dumps(prefs), user_id)
        )
        if notify:
            self.notify(user_id, prefs)
```

**✅ What Should Happen**

```python
def save_preferences(db, user_id, preferences):
    db.execute(
        'UPDATE users SET preferences = ? WHERE id = ?',
        (json.dumps(preferences), user_id)
    )
```

If later someone asks for validation or merging, add it then. Not before.

---

## 3. Surgical Changes

### Example: Fix the bug, not the neighborhood

**User Request:** "Fix the bug where empty emails crash the validator"

**❌ What LLMs Do (Rewrite the file while they are there)**

```diff
 def validate_user(user_data):
-    if not user_data['email']:
+    email = user_data.get('email', '').strip()
+    if not email:
         raise ValueError('Email required')
-    if '@' not in user_data['email']:
+    if '@' not in email:
         raise ValueError('Invalid email')
-    if not user_data['username']:
+    if not user_data.get('username'):
         raise ValueError('Username required')
-    return True
+
+    return True
```

**✅ What Should Happen**

```diff
 def validate_user(user_data):
-    if not user_data['email']:
+    email = user_data.get('email', '')
+    if not email or not email.strip():
         raise ValueError('Email required')
+    if '@' not in email:
+        raise ValueError('Invalid email')
```

Only fix the broken part. Leave the rest alone.

---

## 4. Show Me the Code

### Example: "Make it faster" without guessing

**User Request:** "Make the search faster"

**❌ What LLMs Do (Pick a random optimization and hope)**

```python
@lru_cache(maxsize=1000)
async def search(query):
    # 200 lines of caching, async wrappers, and index speculation
    pass
```

**✅ What Should Happen**

```
"Faster" could mean different things:

1. Lower latency
2. Higher throughput
3. Better perceived speed

Current search time: 480ms.
Most useful next step: benchmark the hot path, then optimize the part that actually hurts.
```

**Rule:** no patch until there is a measurable target.

---

## 5. More LKML-Style Failure Modes

### Example: Voodoo Programming

**User Request:** "Fix the race condition"

**❌ What AI Often Does (cargo-cult garbage)**

```c
spin_lock(&lock);
barrier();
barrier();
retry = 3;
while (retry--) {
    if (state_ready())
        break;
    cpu_relax();
}
spin_unlock(&lock);
```

**Review:**

- This is voodoo programming.
- Random barriers are not a design.
- Random retry loops are not a correctness argument.
- Show the race, explain the ordering, or stop adding garbage.

**✅ What Should Happen**

```
1. Reproduce the race with a test or trace
2. Identify the actual ordering bug
3. Fix the synchronization rule directly
4. Verify the race is gone
```

### Example: Hack Upon Hack

**User Request:** "Make this legacy path work"

**❌ What AI Often Does**

```python
if is_legacy_mode:
    if config.get('legacy_v2'):
        return old_handler_v2(data)
    if config.get('legacy_fallback'):
        return old_handler(data)
    return old_handler_compat(data)
```

**Review:**

- This is hack upon hack.
- You're institutionalizing half-assery instead of fixing the model.
- If the old path is broken, either fix it or isolate it. Do not let the disease spread.

### Example: Pointless Merge Crap

**User Request:** "Prepare a clean PR"

**❌ What AI Often Does**

- rewrites history for no reason
- merges main into the feature branch three times in one day
- includes unrelated formatting changes
- ships commit messages like `misc fixes` or `cleanup`

**Review:**

- This is pointless merge crap.
- Your merge message sucks.
- Explain what changed and why, or don't waste anybody's time.

### Example: Do Not Break Userspace

**User Request:** "Simplify the API by removing deprecated support"

**❌ What AI Often Does**

```js
// removed old endpoint because new endpoint is cleaner
app.delete('/api/v1/export')
```

**Review:**

- No. We do not break userspace because the new thing feels cleaner to you.
- Existing consumers matter.
- "Users should just migrate" is not an argument.

**✅ What Should Happen**

```js
app.get('/api/v1/export', legacyExportHandler)
app.get('/api/v2/export', newExportHandler)
```

Or keep compatibility until the user explicitly asks to break it.

### Example: Too Ugly to Live

**❌ What AI Often Does**

```c
if ((a && !b) || (a && c && !d) || (!a && b && e) ||
    (flag && mode == X && ((t && !u) || (v && w && !z)))) {
    do_the_thing();
}
```

**Review:**

- This patch makes my eyes bleed.
- This is too ugly to live.
- Fix the data flow or split the logic into sane pieces.
