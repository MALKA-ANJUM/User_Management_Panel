# Architecture Diagrams & Visual Flow

## 1. Backend Request-Response Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                         FRONTEND (React)                             │
│  - User submits form                                                 │
│  - Axios attaches Bearer token                                       │
│  - Sends HTTP request                                                │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    LARAVEL API LAYER                                │
│                   (api.php routes)                                   │
│  - Route matching                                                    │
│  - Middleware processing (auth:sanctum, role middleware)            │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    HTTP CONTROLLERS                                  │
│  - Request validation                                                │
│  - Input sanitization                                                │
│  - Dependency injection of Service                                   │
│  - Return JSON response                                              │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    SERVICE LAYER                                     │
│  - Business logic implementation                                     │
│  - Data transformation                                               │
│  - Caching decisions                                                 │
│  - Logging operations                                                │
│  - Error handling                                                    │
│  - Dependency injection of Repository                                │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    REPOSITORY LAYER                                  │
│  - AbstractData Access Logic                                         │
│  - Query building                                                    │
│  - Pagination/Sorting                                                │
│  - No business logic                                                 │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                 ELOQUENT MODELS & ORM                                │
│  - Model definitions                                                 │
│  - Relationships                                                     │
│  - Mass assignment protection                                        │
├─────────────────────────────────────────────────────────────────────┤
│                      DATABASE                                        │
│  - Tables (users, personal_access_tokens, roles, permissions)       │
└─────────────────────────────────────────────────────────────────────┘
```

## 2. Component Hierarchy - Frontend

```
App.jsx (Root)
│
├── Provider: BrowserRouter
│   └── Provider: ToastContainer
│
└── Routes:
    ├── Public Routes:
    │   ├── "/" → Dashboard
    │   ├── "/login" → Login Page
    │   │   └── Dispatches: setToken, setRole
    │   └── "/register" → Register Page
    │
    ├── Protected Routes (ProtectedRoute HOC):
    │   ├── "/user" → User Management (Admin only)
    │   │   ├── Fetches paginated user list
    │   │   ├── Edit Modal (UserModal)
    │   │   └── Delete with confirmation
    │   │
    │   └── "/user/profile" → UserProfile (User/Admin)
    │       ├── Fetches own profile
    │       └── Edit Modal (UserModal)
    │
    └── Layout Components:
        ├── Header (Across all pages)
        │   ├── Logo/Brand
        │   └── Conditional: Login/Register/Logout buttons
        │
        └── Footer (Across all pages)
```

## 3. Dependency Injection Flow - Backend

```
┌──────────────────────────────────────────────────────┐
│        AppServiceProvider.php (register method)       │
│  Binds interfaces to concrete implementations:        │
│                                                        │
│  - UserRepositoryInterface → UserRepository           │
│  - AuthRepositoryInterface → AuthRepository           │
└──────────────────────────────────────────────────────┘
                      │
                      ▼
┌──────────────────────────────────────────────────────┐
│           Laravel Service Container                   │
│  (Singleton instances available throughout app)      │
└──────────────────────────────────────────────────────┘
                      │
        ┌─────────────┼─────────────┐
        │             │             │
        ▼             ▼             ▼
┌────────────────┐ ┌────────────┐ ┌──────────────┐
│  Controller    │ │  Service   │ │  Repository  │
│  public        │ │  injection │ │  injection   │
│  __construct   │ │            │ │              │
│  (Service $s)  │ │ __construct│ │ __construct  │
│  {             │ │ (Interface)│ │ (Model)      │
│  $this->s = $s;│ │ {          │ │ {            │
│  }             │ │ $this->r=$r│ │ $this->m=$m; │
│                │ │ }          │ │ }            │
└────────────────┘ └────────────┘ └──────────────┘
```

Result: Loosely coupled, testable, maintainable code.

## 4. Authentication & Authorization Pipeline

```
Request Headers: Authorization: Bearer <token>
                          │
                          ▼
              ┌─────────────────────────┐
              │  auth:sanctum middleware │
              └─────────────────────────┘
                      │
        ┌─────────────┴─────────────┐
        │ Token valid?              │ Token invalid?
        │ Exists in DB?             │
        ▼                           ▼
    YES                         THROW 401 Unauthorized
        │
        ▼
    ┌────────────────────────────┐
    │ Load Associated User Model  │
    │ request()->user()           │
    └────────────────────────────┘
        │
        ▼
    ┌────────────────────────────┐
    │  role:admin middleware      │  (if applied)
    │  Check user.roles          │
    └────────────────────────────┘
        │
    ┌───┴───┐
    │       │
  YES(✓)   NO(✗)
    │       │
    │       └─→ THROW 403 Forbidden
    │
    ▼
┌────────────────────────────┐
│  Handle request in logic    │
│  Access request()->user()   │
└────────────────────────────┘
    │
    ▼
┌────────────────────────────┐
│  Return JSON response       │
│  with appropriate data      │
└────────────────────────────┘
```

## 5. Token-Based Authentication Flow - Frontend

```
User Action: Submit Form (Login/Register)
                  │
                  ▼
        ┌─────────────────────┐
        │  axiosClient.post() │
        │  to /api/login      │
        │  or /api/register   │
        └─────────────────────┘
                  │
                  ▼
        ┌─────────────────────────────────────┐
        │  Axios Request Interceptor          │
        │  Get token from localStorage        │
        │  Add Authorization header:          │
        │  "Bearer <token>"                   │
        └─────────────────────────────────────┘
                  │
                  ▼
        ┌─────────────────────┐
        │  Backend validates  │
        │  Returns {          │
        │    token,           │
        │    user,            │
        │    role             │
        │  }                  │
        └─────────────────────┘
                  │
                  ▼
        ┌─────────────────────────────────┐
        │  Frontend receives response      │
        │  const { data } = await axios   │
        │  setToken(data.token)           │
        │  setRole(data.role)             │
        └─────────────────────────────────┘
                  │
                  ▼
        ┌─────────────────────────────────┐
        │  Save to localStorage &         │
        │  update React state             │
        │  localStorage.setItem(...)      │
        └─────────────────────────────────┘
                  │
                  ▼
        ┌─────────────────────────────────┐
        │  Role-based navigation:         │
        │  if role === 'admin'            │
        │    → navigate('/user')          │
        │  else                           │
        │    → navigate('/user/profile')  │
        └─────────────────────────────────┘
```

## 6. Background Job & Queue Flow

```
Event Trigger: 
- Command Schedule (every 5 minutes)
- Manual dispatch

                  │
                  ▼
┌──────────────────────────────────────┐
│  "users:send-mail" command executed  │
│  SendScheduledUserMailCommand        │
└──────────────────────────────────────┘
                  │
                  ▼
┌──────────────────────────────────────┐
│  SendUserMailJob::dispatch()         │
│  (Push to queue - Redis/Database)    │
└──────────────────────────────────────┘
                  │
                  ▼
┌──────────────────────────────────────┐
│  Queue Worker Listening              │
│  `php artisan queue:work`            │
│  Picks up job from queue             │
└──────────────────────────────────────┘
                  │
                  ▼
┌──────────────────────────────────────┐
│  SendUserMailJob::handle()           │
│  Dependency Injection: UserService   │
└──────────────────────────────────────┘
                  │
                  ▼
┌──────────────────────────────────────┐
│  UserService::getUsersForMail()      │
│  Returns cursor for memory efficiency│
└──────────────────────────────────────┘
                  │
                  ▼
┌──────────────────────────────────────┐
│  Chunk users (100 at a time)         │
│  foreach user:                       │
│    Mail::to($user)->queue(...)       │
└──────────────────────────────────────┘
                  │
                  ▼
┌──────────────────────────────────────┐
│  Mail events queued                  │
│  Queue worker processes mail events  │
└──────────────────────────────────────┘
                  │
                  ▼
┌──────────────────────────────────────┐
│  SendUserMail::build()               │
│  Email sent to each user             │
└──────────────────────────────────────┘
                  │
                  ▼
┌──────────────────────────────────────┐
│  Success logged                      │
│  Failed attempts retried (3x default)│
└──────────────────────────────────────┘
```

## 7. Task Scheduler Flow

```
Crontab:
* * * * * cd /app && php artisan schedule:run >> /dev/null 2>&1

(Runs every minute)

                  │
                  ▼
└──────────────────────────────────────┐
│  Kernel::schedule() method called    │
└──────────────────────────────────────┘
                  │
        ┌─────────┴─────────┐
        │                   │
        ▼                   ▼
    ┌──────────────┐   ┌──────────────────┐
    │ Every 5 min? │   │ At 02:00 AM?     │
    │ Execute:     │   │ Execute:         │
    │ users:       │   │ cleanup:         │
    │ send-mail    │   │ daily            │
    └──────────────┘   └──────────────────┘
        │                   │
        ▼                   ▼
    ┌──────────────┐   ┌──────────────────┐
    │ Dispatch:    │   │ Delete expired   │
    │ SendUserMail │   │ tokens from DB   │
    │ Job          │   │ Delete logs >30  │
    │              │   │ days old         │
    └──────────────┘   └──────────────────┘
        │                   │
        └─────────┬─────────┘
                  ▼
┌──────────────────────────────────────────┐
│  withoutOverlapping():                   │
│  Prevents duplicate execution if         │
│  previous run still in progress          │
└──────────────────────────────────────────┘
```

## 8. RBAC (Role-Based Access Control) Model

```
USER MODEL (Spatie Permission)
│
├── Roles:
│   ├── admin
│   │   └── Permissions: [manage-users, view-reports]
│   │
│   └── user
│       └── Permissions: [view-profile, edit-profile]
│
└── Direct Permissions: (optional)


ROUTE PROTECTION
│
├── Public Routes:
│   ├── GET   /login        (No auth required)
│   └── POST  /register     (No auth required)
│
└── Protected Routes:
    │
    ├── Base: auth:sanctum middleware
    │   └── Validates Sanctum token
    │
    └── Role-based: role:admin | role:user middleware
        ├── GET    /user           → role:admin
        ├── POST   /user/store     → role:admin
        ├── DELETE /user/delete    → role:admin
        └── GET    /user/profile   → role:user | role:admin


ENFORCEMENT IN LAYERS
1. Route Middleware (first line of defense)
2. Service Layer (business rules)
3. Repository (data visibility)
4. Policy (fine-grain authorization)
```

## 9. Data Model Relationships

```
┌─────────────────────────┐
│      USERS TABLE        │
├─────────────────────────┤
│ id (PK)                 │
│ name                    │
│ email (UNIQUE)          │
│ phone                   │
│ gender                  │
│ address                 │
│ password (HASHED)       │
│ created_at              │
│ updated_at              │
└─────────────────────────┘
         │
    ┌────┴────┐
    │          │
    ▼          ▼
┌───────────────────────┐   ┌──────────────────────────┐
│ PERSONAL_ACCESS_      │   │  MODEL_HAS_ROLES         │
│ TOKENS TABLE (JWT)    │   │  (Spatie)                │
├───────────────────────┤   ├──────────────────────────┤
│ id (PK)               │   │ role_id (FK)             │
│ tokenable_id (FK)     │   │ model_id (FK)            │
│ name                  │   │ model_type               │
│ token (HASHED)        │   └──────────────────────────┘
│ expires_at            │            │
│ created_at            │            ▼
└───────────────────────┘   ┌──────────────────────────┐
         │                  │  ROLES TABLE (Spatie)    │
         │                  ├──────────────────────────┤
         │                  │ id (PK)                  │
         │                  │ name (admin, user)       │
         │                  │ guard_name               │
         │                  └──────────────────────────┘
                                   │
                                   ▼
                            ┌──────────────────────────┐
                            │  ROLE_HAS_PERMISSIONS    │
                            │  (Spatie)                │
                            └──────────────────────────┘
                                   │
                                   ▼
                            ┌──────────────────────────┐
                            │  PERMISSIONS TABLE       │
                            │  (Spatie)                │
                            └──────────────────────────┘
```

## 10. Error Handling Flow

### Backend
```
User Request
     │
     ▼
Try-Catch in Controller/Service
     │
     ├─ Validation Exception
     │  └─ Return 422 + validation errors
     │
     ├─ Model Not Found
     │  └─ Return 404 + message
     │
     ├─ Authorization Failed
     │  └─ Return 403 + message
     │
     ├─ Database Query Error
     │  └─ Log error, return 500
     │
     └─ Unexpected Exception
        └─ Log error, return 500 generic message
```

### Frontend
```
Axios POST/GET request
     │
     ├─ Success (2xx)
     │  └─ Process response
     │     └─ Update state
     │     └─ Show success toast
     │
     └─ Error (4xx/5xx)
        └─ Extract error message from response
           └─ Show error toast
           └─ Handle redirect if needed (401→login)
```

