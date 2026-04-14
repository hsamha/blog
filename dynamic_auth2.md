[Back](./README.md)

# 🚀 How to Implement Real-Time Permission Updates with ASP.NET, Angular & SignalR

## 🧠 Overview

This solution provides:

* Lightweight JWT (no permissions inside)
* Permissions loaded on Angular bootstrap
* Real-time updates via SignalR
* Updates sent **only to the affected user**
* Backend enforcement via a **custom attribute (supports multiple roles/permissions)**

---

## 🏗️ Architecture Flow

1. User logs in → receives JWT (userId + basic claims)
2. Angular boots → fetches permissions
3. Permissions cached client-side
4. Admin updates permissions
5. Backend pushes `permissions_updated` via SignalR
6. Only the specific user receives it
7. Angular reloads permissions
8. UI updates instantly

---

# 📡 Backend (ASP.NET Core)

---

## 1. Install SignalR

```bash
dotnet add package Microsoft.AspNetCore.SignalR
```

---

## 2. 🔐 Ensure SignalR Uses Authenticated Connections

This is **critical** so SignalR knows which user is connected.

### Step A — Configure JWT Authentication

```csharp
builder.Services.AddAuthentication("Bearer")
    .AddJwtBearer("Bearer", options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            // your validation params
        };

        // 🔥 IMPORTANT: Allow SignalR to read token from query string
        options.Events = new JwtBearerEvents
        {
            OnMessageReceived = context =>
            {
                var accessToken = context.Request.Query["access_token"];

                var path = context.HttpContext.Request.Path;

                if (!string.IsNullOrEmpty(accessToken) &&
                    path.StartsWithSegments("/hubs/permissions"))
                {
                    context.Token = accessToken;
                }

                return Task.CompletedTask;
            }
        };
    });
```

---

### Step B — Enable Authentication & Authorization

```csharp
app.UseAuthentication();
app.UseAuthorization();
```

---

### Step C — Secure the Hub

```csharp
[Authorize]
public class PermissionsHub : Hub
{
    public override async Task OnConnectedAsync()
    {
        var userId = Context.User?.FindFirst("userId")?.Value;

        if (!string.IsNullOrEmpty(userId))
        {
            await Groups.AddToGroupAsync(Context.ConnectionId, userId);
        }

        await base.OnConnectedAsync();
    }
}
```

✅ Now SignalR connections are **authenticated using the same JWT**

---

## 3. Configure SignalR Endpoint

```csharp
builder.Services.AddSignalR();

app.MapHub<PermissionsHub>("/hubs/permissions");
```

---

## 4. Push Event When Permissions Change

```csharp
public class PermissionService
{
    private readonly IHubContext<PermissionsHub> _hubContext;

    public PermissionService(IHubContext<PermissionsHub> hubContext)
    {
        _hubContext = hubContext;
    }

    public async Task UpdateUserPermissions(string userId, List<string> permissions)
    {
        // Update DB
        // ...

        // Notify ONLY this user
        await _hubContext.Clients.Group(userId)
            .SendAsync("permissions_updated");
    }
}
```

---

## 5. Permissions API

```csharp
[Authorize]
[ApiController]
[Route("api/permissions")]
public class PermissionsController : ControllerBase
{
    [HttpGet]
    public IActionResult GetPermissions()
    {
        var userId = User.FindFirst("userId")?.Value;

        var permissions = new List<string>
        {
            "VIEW_DASHBOARD",
            "EDIT_USER"
        };

        return Ok(permissions);
    }
}
```

---

## 6. ✅ Custom Attribute (Supports Multiple Permissions/Roles)

### ✔ Updated to accept multiple values

```csharp
public class HasPermissionAttribute : Attribute, IAsyncAuthorizationFilter
{
    private readonly string[] _permissions;

    public HasPermissionAttribute(params string[] permissions)
    {
        _permissions = permissions;
    }

    public async Task OnAuthorizationAsync(AuthorizationFilterContext context)
    {
        var userId = context.HttpContext.User.FindFirst("userId")?.Value;

        var userPermissions = GetUserPermissions(userId);

        // ✅ Check if user has ANY of the required permissions
        var hasPermission = _permissions.Any(p => userPermissions.Contains(p));

        if (!hasPermission)
        {
            context.Result = new ForbidResult();
        }
    }

    private List<string> GetUserPermissions(string userId)
    {
        // Replace with DB/cache
        return new List<string> { "VIEW_DASHBOARD" };
    }
}
```

---

### ✔ Usage Examples

```csharp
[HasPermission("EDIT_USER", "ADMIN")]
[HttpPost("edit")]
public IActionResult EditUser()
{
    return Ok();
}
```

```csharp
[HasPermission("VIEW_DASHBOARD")]
[HttpGet("dashboard")]
public IActionResult Dashboard()
{
    return Ok();
}
```

---

## 🌐 Frontend (Angular)

---

## 1. Install SignalR

```bash
npm install @microsoft/signalr
```

---

## 2. Permission Service

```ts
@Injectable({ providedIn: 'root' })
export class PermissionService {
  private permissions: string[] = [];

  constructor(private http: HttpClient) {}

  loadPermissions(): Observable<string[]> {
    return this.http.get<string[]>('/api/permissions').pipe(
      tap(perms => this.permissions = perms)
    );
  }

  hasPermission(permission: string): boolean {
    return this.permissions.includes(permission);
  }
}
```

---

## 3. Load Permissions on Bootstrap

```ts
export function initPermissions(permissionService: PermissionService) {
  return () => permissionService.loadPermissions().toPromise();
}
```

---

## 4. 🔐 Pass Token to SignalR (Authenticated Connection)

```ts
startConnection(token: string) {
  this.hubConnection = new signalR.HubConnectionBuilder()
    .withUrl('/hubs/permissions', {
      accessTokenFactory: () => token
    })
    .withAutomaticReconnect()
    .build();

  this.hubConnection.start();

  this.hubConnection.on('permissions_updated', () => {
    this.permissionService.loadPermissions().subscribe();
  });
}
```

✅ This ensures the backend receives the JWT

---

## 5. Start SignalR

```ts
constructor(signalRService: SignalRService, authService: AuthService) {
  const token = authService.getToken();
  signalRService.startConnection(token);
}
```

---

## 6. Use Permissions in UI

```html
<button *ngIf="permissionService.hasPermission('EDIT_USER')">
  Edit User
</button>
```

---

# 🔐 Security Best Practices

* ✅ Always validate permissions in backend
* ✅ Never trust frontend-only checks
* ✅ Use caching (Redis recommended)
* ✅ Keep JWT small (no permissions inside)
* ✅ Use HTTPS (required for SignalR in production)

---

# 📊 Sequence Diagram (Conceptual)

```
User → Login → JWT (no permissions)

Angular → GET /permissions → Cache

Admin → Update permissions

Backend → SignalR → permissions_updated → Specific User

Angular → GET /permissions → Update UI
```

---

# ✅ Final Result

You now have:

* 🔄 Real-time permission updates
* 🎯 Per-user targeting (not broadcast)
* 🔐 Fully authenticated SignalR
* 🧩 Flexible multi-permission authorization
* ⚡ Clean and scalable design

---

Here’s a **practical testing + analysis guide** you can attach to your article so others can confidently validate and reason about the solution.

---

# 🧪 How to Test Real-Time Permission Updates (Manual + Edge Cases + Trade-offs)

---

## ✅ 1. Manual Test Cases (Step-by-Step)

### 🔐 Test Case 1 — Initial Permission Load on Login

**Goal:** Ensure permissions are fetched after login

**Steps:**

1. Login as User A
2. Open browser dev tools → Network tab
3. Observe request to:

   ```
   GET /api/permissions
   ```
4. Verify response contains expected permissions
5. Confirm UI reflects permissions (buttons, routes, etc.)

**Expected Result:**

* Permissions are loaded once on app bootstrap
* UI renders correctly

---

### 🔄 Test Case 2 — Real-Time Update (Core Scenario)

**Goal:** Verify SignalR pushes update and UI refreshes

**Steps:**

1. Login as User A (keep app open)
2. Login as Admin in another session
3. Change User A permissions (e.g., remove `EDIT_USER`)
4. Observe User A browser

**Expected Result:**

* SignalR event `permissions_updated` is received
* Angular calls `/api/permissions` again
* UI updates immediately (e.g., button disappears)

---

### 🎯 Test Case 3 — Ensure Only Target User Gets Update

**Steps:**

1. Login as User A and User B in separate browsers
2. Admin updates **User A permissions only**
3. Monitor both browsers

**Expected Result:**

* User A receives update ✅
* User B receives NOTHING ❌

---

### 🔌 Test Case 4 — Reconnection Handling

**Goal:** Ensure system recovers from connection loss

**Steps:**

1. Login as User A
2. Disable internet (or stop backend temporarily)
3. Re-enable connection
4. Admin updates permissions during/after reconnect

**Expected Result:**

* SignalR reconnects automatically
* Event still received OR next refresh fixes state

---

### 🧱 Test Case 5 — Backend Enforcement (Critical Security)

**Steps:**

1. Login as User A
2. Remove permission via Admin
3. Try calling protected API manually (Postman)

**Expected Result:**

* API returns `403 Forbidden`
* Even if UI still shows button (temporary lag)

---

### 🧪 Test Case 6 — Multiple Permissions Attribute

**Steps:**

1. Protect endpoint with:

   ```csharp
   [HasPermission("ADMIN", "EDIT_USER")]
   ```
2. Test with users having:

   * Only ADMIN
   * Only EDIT_USER
   * Neither

**Expected Result:**

* Access granted if **ANY** permission matches

---

### 🔁 Test Case 7 — Token Expiry + SignalR

**Steps:**

1. Login with short-lived token
2. Wait until token expires
3. Trigger permission update

**Expected Result:**

* SignalR connection fails or reconnects with new token
* Requires refresh or re-auth

---

### 📱 Test Case 8 — Multiple Tabs

**Steps:**

1. Open same user in 2 tabs
2. Update permissions

**Expected Result:**

* Both tabs receive update
* Both refresh permissions

---

## ⚠️ 2. Edge Cases You Must Consider

---

### ⏱️ 1. Race Condition (UI vs Backend)

**Scenario:**

* Admin removes permission
* User clicks button before UI updates

**Risk:**

* UI still shows access

**Mitigation:**

* Backend always enforces permissions ✅ (already done)

---

### 🔌 2. SignalR Connection Not Established

**Scenario:**

* User loads app but SignalR fails

**Impact:**

* No real-time updates

**Mitigation:**

* Periodic refresh (optional fallback)
* Show connection status (optional)

---

### 🔄 3. Missed Events

**Scenario:**

* User offline during update

**Impact:**

* No event received

**Mitigation:**

* Reload permissions on:

  * reconnect
  * route change (optional)
  * manual refresh

---

### 🔐 4. Token Expiration During Connection

**Scenario:**

* JWT expires while SignalR is active

**Impact:**

* Hub disconnects silently

**Mitigation:**

* Reconnect with fresh token
* Integrate with auth refresh flow

---

### 🧠 5. Stale Permission Cache

**Scenario:**

* Frontend cache not refreshed

**Mitigation:**

* Always reload after SignalR event (already implemented)

---

### 👥 6. User Connected Multiple Times

**Scenario:**

* Same user logs in from multiple devices

**Behavior:**

* All connections in same group receive update ✅

---

### 📦 7. High Frequency Updates

**Scenario:**

* Admin updates permissions repeatedly

**Impact:**

* Many API calls from frontend

**Mitigation:**

* Debounce updates (frontend)
* Send payload instead of forcing refetch (advanced)

---

## ⚖️ 3. Trade-offs of This Approach

---

### ✅ Advantages

#### 🔄 Real-Time UX

* Immediate UI updates without refresh

#### 🪶 Lightweight JWT

* Token stays small and fast

#### 🎯 Targeted Updates

* Only affected users notified

#### 🔐 Strong Security

* Backend enforces permissions independently

---

### ❌ Trade-offs / Costs

---

#### 1. ⚙️ Increased Complexity

* SignalR setup + lifecycle management
* More moving parts vs simple polling

---

#### 2. 🔌 Persistent Connection Overhead

* Each user maintains open connection
* Can impact scalability

👉 Mitigation:

* Use Azure SignalR / Redis backplane

---

#### 3. 📡 Dependency on Real-Time Channel

* If SignalR fails → no updates

👉 Mitigation:

* Add fallback polling (optional)

---

#### 4. 🔁 Extra API Calls

* Every update triggers:

  ```
  SignalR → API call → UI update
  ```

👉 Optimization:

* Send updated permissions in SignalR payload (advanced)

---

#### 5. 🔐 Token Handling Complexity

* SignalR requires token injection + refresh handling

---

#### 6. 📊 Horizontal Scaling Challenges

If you scale backend:

* Users connected to different servers
* SignalR needs shared state

👉 Solution:

* Redis backplane
* Azure SignalR Service

---

## 🧠 4. When This Approach Is Ideal

Use this pattern when:

* Permissions change **frequently**
* UX must be **real-time**
* You want **fine-grained access control**
* You support **multiple active sessions per user**

---

## 🚫 When NOT to Use It

Avoid if:

* Permissions rarely change
* Simpler polling is acceptable
* System scale is very small
* You want minimal infrastructure

---

## ✅ Final Checklist Before Production

* [ ] SignalR authentication works
* [ ] Backend attribute enforced everywhere
* [ ] Permissions cached (Redis recommended)
* [ ] Token refresh handled
* [ ] Reconnection tested
* [ ] Multi-device scenario tested
* [ ] Load tested with many connections

---

## 🎯 Final Thought

This pattern is **powerful but not free**:

> You’re trading **simplicity** for **real-time accuracy and UX**

If implemented correctly, it gives you a **robust, scalable, and secure permission system** used in enterprise-grade apps.

---
[Back](./README.md)
