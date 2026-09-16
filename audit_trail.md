# Building an Automatic Audit Trail in .NET with EF Core

*Who changed what, when, and from where, without touching a single handler.*

---

## The problem

Every business application eventually hears the same question:

> "Who changed this?"

A price changed. A user lost a permission. A currency was deleted. Someone needs to know what happened.

Many projects start with columns like these on each table:

```csharp
public DateTime CreatedAtUtc { get; set; }
public string? CreatedBy { get; set; }
public DateTime? ModifiedAtUtc { get; set; }
public string? ModifiedBy { get; set; }
```

That's useful, but it only tells you the **last** person who touched a row. You lose everything before that. You also don't know **what** changed.

We wanted a real history that works like this:

- Every insert, update and delete is recorded.
- Each record says who did it and when.
- We can see which fields changed, with old and new values.
- History can be shown on any screen: roles, users, currencies, payment methods, and so on.
- Developers **don't have to remember** to write logging code.

This article explains how we designed it.

---

## The big idea: let EF Core do the watching

Our app is modular. Each module has its own `DbContext`, and every one of them inherits from a shared base class:

```csharp
public abstract class ModuleDbContext : DbContext { ... }

public sealed class SystemDbContext : ModuleDbContext { ... }
public sealed class AccountingDbContext : ModuleDbContext { ... }
```

Every save in the application goes through `SaveChangesAsync`. EF Core's **ChangeTracker** already knows exactly what is about to be saved: which rows were added, modified or deleted, and the original and current value of every property.

So we don't need logging calls in our business code. We only need to **read the ChangeTracker in one place**, just before saving.

```
Handler changes entities
        │
        ▼
ModuleDbContext.SaveChangesAsync()
        │
        ├── 1. Read ChangeTracker → build audit entries
        ├── 2. Save the real data
        └── 3. Save the audit entries (same transaction)
```

We made three decisions early:

1. **Every entity is audited by default.** There's no opt-in interface. If you have to remember to opt in, someone will forget.
2. **Data and audit logs are saved in the same transaction.** If the save fails, no audit row is written. If the audit fails, the data is not saved either.
3. **No "empty" history.** If someone clicks *Save* without changing anything, we write nothing.

---

## The tables

We use two small tables.

### `AuditLogs`: one row per change

```csharp
public class AuditLog
{
    public long Id { get; set; }
    public int TenantId { get; set; }

    public string EntityType { get; set; } = "";   // "Currency"
    public string EntityKey { get; set; } = "";    // "7", or "5|12" for composite keys

    public AuditAction Action { get; set; }        // Created, Updated, Deleted, Restored
    public string Changes { get; set; } = "";      // JSON list of field changes

    public string? UserId { get; set; }
    public DateTime OccurredAtUtc { get; set; }
    public Guid CorrelationId { get; set; }        // groups changes from one save
    public string? ScreenRef { get; set; }         // the screen the change came from
}

public enum AuditAction { Created, Updated, Deleted, Restored }
```

A few notes:

- **`EntityKey` is a string.** Most tables have an `int` id, but junction tables such as `RolePermission` often use a composite key. A string handles both: `"7"` or `"5|12"`.
- **`Changes` is JSON.** It keeps the table simple and works for any entity shape.
- **`CorrelationId`** groups everything that happened in one save. If a user saves a role and three permissions at once, all four log rows share the same id.

Here is an example of `Changes` for an update:

```json
[
  { "field": "Name",   "old": "US Dollar", "new": "U.S. Dollar" },
  { "field": "Symbol", "old": "USD",       "new": "$" }
]
```

### `AuditLogParents`: which records a change belongs to

We'll explain this table in the next section.

```csharp
public class AuditLogParent
{
    public long AuditLogId { get; set; }
    public string ParentType { get; set; } = "";   // "Role"
    public string ParentKey { get; set; } = "";    // "5"
}
```

---

## The tricky part: child tables

Imagine a role called **Cashier** with id `5`. You open its *Activity* tab and expect to see:

1. Role created
2. Name changed from "Cashier" to "Senior Cashier"
3. Permission "Delete Invoice" added

Items 1 and 2 are changes to the `Roles` table. Item 3 is not. Adding a permission to a role inserts a row into the `RolePermissions` table:

| EntityType     | EntityKey | Action  |
|----------------|-----------|---------|
| Role           | 5         | Created |
| Role           | 5         | Updated |
| RolePermission | 5\|12     | Created |

If the Activity tab only asks for *"EntityType = Role and EntityKey = 5"*, the permission change is **missing**. That's a real problem, because it's often the most important change.

### Solution: record the parent too

For every log row, we also record which "parent" records it belongs to:

| AuditLog                    | ParentType | ParentKey |
|-----------------------------|------------|-----------|
| RolePermission 5\|12 Created | Role       | 5         |

Now the Role 5 Activity tab asks:

> Give me logs **for Role 5**, *or* logs **whose parent is Role 5**.

All three items appear.

### How do we know who the parent is?

We could ask developers to mark each child class:

```csharp
// We did NOT do this
public class RolePermission : IAuditChild
{
    public string ParentEntityType => "Role";
    public int ParentEntityId => RoleId;
}
```

It works, but it's one more thing to remember. And EF Core **already knows** these relationships, because we configured them.

So we read them from the EF model instead. The rule is simple:

> A foreign key points to a **parent** when the principal entity **navigates to** this child: a collection (`Role.RolePermissions`) or a one-to-one reference (`User.Restrictions`).

```csharp
public class Role
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public ICollection<RolePermission> RolePermissions { get; set; } = []; // ← this collection
}

public class RolePermission
{
    public int RoleId { get; set; }        // ← FK to Role  → Role is a parent
    public int PermissionId { get; set; }
    public Role Role { get; set; } = null!;
}
```

In code, it looks roughly like this:

```csharp
private static IEnumerable<(string ParentType, string ParentKey)> FindParents(EntityEntry entry)
{
    foreach (var fk in entry.Metadata.GetForeignKeys())
    {
        // Does the principal have a navigation (collection or one-to-one) pointing to us?
        var isParent = fk.PrincipalToDependent is not null;
        if (!isParent) continue;

        var values = fk.Properties
            .Select(p => entry.State == EntityState.Deleted
                ? entry.OriginalValues[p]
                : entry.CurrentValues[p]);

        yield return (fk.PrincipalEntityType.ClrType.Name, string.Join("|", values));
    }
}
```

We cache this per entity type, because the EF model never changes at runtime.

### A nice bonus: a change can have many parents

Take `UserRole`, the table that connects users and roles. It has two foreign keys, and both `User` and `Role` have a collection of `UserRoles`. So when we assign **Cashier** to **Ahmad**, the change is linked to both:

| AuditLog          | ParentType | ParentKey |
|-------------------|------------|-----------|
| UserRole Created  | User       | 42        |
| UserRole Created  | Role       | 5         |

It appears in **Ahmad's** activity ("Role Cashier assigned") **and** in the **Cashier** role's activity ("Assigned to Ahmad"). We get that for free.

### What about lookups?

An `Invoice` may have a `CurrencyId`. We don't want every invoice to flood the Currency's activity tab. Because `Currency` has no `Invoices` navigation, the rule above ignores it. If you ever want a child to show under a parent, add the navigation to the parent and it works.

---

## Capturing the changes step by step

Here is the flow inside `SaveChangesAsync`.

### Step 1: take a snapshot *before* anything else

Our base context already converts deletes into **soft deletes** (it sets `IsDeleted = true` instead of removing the row). That turns a `Deleted` entry into a `Modified` one. So we must look at the ChangeTracker **first**, while we can still tell what really happened.

```csharp
public override async Task<int> SaveChangesAsync(CancellationToken ct = default)
{
    var pending = CaptureAuditEntries();   // 1. snapshot first
    StampEntities();                       // 2. existing: CreatedBy, soft delete, ...
    var result = await base.SaveChangesAsync(ct);

    if (pending.Count > 0)
        await WriteAuditLogsAsync(pending, ct);

    return result;
}
```

### Step 2: decide the action

```csharp
private static AuditAction? ResolveAction(EntityEntry entry)
{
    if (entry.State == EntityState.Added)   return AuditAction.Created;
    if (entry.State == EntityState.Deleted) return AuditAction.Deleted;

    if (entry.State == EntityState.Modified && entry.Entity is ISoftDeletable)
    {
        var prop = entry.Property(nameof(ISoftDeletable.IsDeleted));
        var wasDeleted = (bool)prop.OriginalValue!;
        var isDeleted  = (bool)prop.CurrentValue!;

        if (!wasDeleted && isDeleted) return AuditAction.Deleted;
        if (wasDeleted && !isDeleted) return AuditAction.Restored;
    }

    return entry.State == EntityState.Modified ? AuditAction.Updated : null;
}
```

### Step 3: build the diff

- **Created:** every property and its value.
- **Updated:** only properties whose value really changed.
- **Deleted:** the last known values.

```csharp
var changes = entry.Properties
    .Where(p => !IsIgnored(p))
    .Where(p => entry.State != EntityState.Modified
             || (p.IsModified && !Equals(p.OriginalValue, p.CurrentValue)))
    .Select(p => new FieldChange(
        p.Metadata.Name,
        entry.State == EntityState.Added ? null : p.OriginalValue,
        entry.State == EntityState.Deleted ? null : p.CurrentValue))
    .ToList();

if (changes.Count == 0)
    continue; // nothing really changed → no history row
```

The `!Equals(...)` check matters. EF can mark a property as modified even when you set it to the same value. Without this check, your history fills up with rows that say *"Name changed from A to A"*.

### Step 4: skip what should never be logged

Some things should never go into an audit table:

```csharp
public class ApplicationUser
{
    public string UserName { get; set; } = "";

    [AuditIgnore]
    public string PasswordHash { get; set; } = "";
}
```

We also always skip the audit-stamp columns (`ModifiedBy`, `ModifiedAtUtc`, …) because they're just noise, and the tenant id.

Some whole tables are noise too, such as login sessions or grid column preferences. For those:

```csharp
[AuditIgnoreEntity]
public class UserSession { ... }
```

Everything else is audited by default.

### Step 5: save the logs after the data

New rows don't have their database-generated `Id` until **after** they are saved. So:

1. Save the real data.
2. Read the new ids from the tracked entries.
3. Add the `AuditLog` and `AuditLogParent` rows.
4. Save again.

Both saves run inside the **same database transaction** (our Unit of Work opens it), so they succeed or fail together. A simple flag stops the second save from auditing the audit rows themselves.

---

## Using screen reference numbers

Our app has a `Screens` table. Every screen has a unique **reference number**, such as `SYS-010` for the Roles screen. We already used it for things like saved grid layouts. It turned out to be very useful for auditing too.

### 1. The frontend asks by screen, not by class name

Instead of this:

```
GET /api/audit-logs?entityType=Role&entityKey=5
```

the frontend sends this:

```
GET /api/audit-logs?screenRef=SYS-010&entityKey=5
```

We added one column to the screen, `AuditEntityType`. The backend looks up the screen and knows:

- which entity to query (`"Role"`),
- which module's audit table to read,
- whether the user has access to that module.

The frontend never needs to know C# class names. Renaming a class doesn't break the UI.

### 2. One permission per screen

Each screen gets its own `ViewActivity` permission. A user can be allowed to see currency history but **not** user or role history. That matters, because audit logs can reveal sensitive information.

### 3. Recording where a change came from

The frontend adds one header to every request:

```
X-Screen-Ref: SYS-010
```

We store it on every audit row. The same user record can be edited from the **Users** screen, the **My Profile** screen, or a **User Roles** screen. Now the history can say:

> Ahmad changed **Email**, from the *My Profile* screen, on 15 Sep 2026 at 10:42

This header is **only for information**. We never use it to make security decisions, because the client controls it.

---

## The result

One reusable frontend component:

```html
<ActivityLog screenRef="SYS-010" entityKey="5" />
```

It shows a timeline like this:

```
15 Sep 2026 10:42  Ahmad    Updated  Role "Cashier"
                            Name: "Cashier" → "Senior Cashier"
                            (from Roles screen)

15 Sep 2026 10:42  Ahmad    Created  Permission "Delete Invoice" granted

14 Sep 2026 09:10  Sara     Created  Role "Cashier"
```

The API response for one item looks like this:

```json
{
  "id": 1042,
  "action": "Updated",
  "entityType": "Role",
  "entityKey": "5",
  "changes": [
    { "field": "Name", "old": "Cashier", "new": "Senior Cashier" }
  ],
  "userId": "42",
  "userName": "Ahmad",
  "occurredAtUtc": "2026-09-15T07:42:10Z",
  "screenRef": "SYS-010",
  "screenName": "Roles",
  "correlationId": "7b9c1e0a-..."
}
```

And the best part: **business code didn't change at all.** A handler that updates a currency still looks like this:

```csharp
currency.Name = request.Name;
currency.Symbol = request.Symbol;
await _unitOfWork.SaveChangesAsync(ct);
```

History is written automatically.

---

## Things to be aware of

No design is perfect. Here are the limits we accepted:

- **Detached updates have no "before" values.** If you attach an entity you didn't load and call `Update()`, EF doesn't know the old values, so no diff is recorded. Load the entity first, then change it.
- **Bulk operations skip the ChangeTracker.** `ExecuteUpdateAsync`, `ExecuteDeleteAsync` and raw SQL are not audited. Avoid them for important data, or log them yourself.
- **Background jobs and seeders have no user.** Their rows have an empty `UserId`, which the UI shows as "System".
- **The table grows.** Plan a retention or archive job once volume becomes large.
- **Values are stored as they are.** A foreign key is stored as `CurrencyId: 3`, not "US Dollar". Names are resolved when reading, not when writing.

---

## Summary

| Goal | How we did it |
|------|---------------|
| Log every change automatically | Read EF Core's ChangeTracker in the base `DbContext` |
| Never forget an entity | Audited by default; opt **out** only for noise |
| Keep data and logs consistent | Same database transaction |
| No empty history | Compare original and current values |
| Keep secrets out | `[AuditIgnore]` on sensitive properties |
| Show child changes on the parent | Detect parents from EF relationships (`AuditLogParents`) |
| One change, many screens | A row can have many parents |
| Simple, safe API for the frontend | Query by screen reference number |
| Fine-grained access | A `ViewActivity` permission per screen |
| Know where a change came from | `X-Screen-Ref` header stored on each row |

The main lesson: **put cross-cutting behaviour in one place that every save already passes through.** EF Core's ChangeTracker gives you almost everything for free. You only need to read it carefully.
