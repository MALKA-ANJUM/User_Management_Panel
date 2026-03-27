# Quick Reference & Implementation Guide

## Table of Contents
1. [File Structure Overview](#file-structure-overview)
2. [API Endpoints Reference](#api-endpoints-reference)
3. [Key Implementation Details](#key-implementation-details)
4. [Development Setup](#development-setup)
5. [Common Patterns Used](#common-patterns-used)
6. [Troubleshooting Guide](#troubleshooting-guide)

---

## File Structure Overview

### Backend Structure
```
backend/
├── app/
│   ├── Console/
│   │   ├── Kernel.php                          ← Schedule definitions
│   │   └── Commands/
│   │       ├── SendScheduledUserMailCommand.php ← Dispatch mail job
│   │       └── DailyCleanup.php                 ← Clean tokens & logs
│   │
│   ├── Http/
│   │   ├── Controllers/
│   │   │   ├── AuthController.php               ← Register, Login, Logout
│   │   │   ├── UserController.php               ← User CRUD, Profile
│   │   │   └── Controller.php                   ← Base controller
│   │   ├── Middleware/                          ← Custom middleware
│   │   └── Kernel.php                           ← Middleware registration
│   │
│   ├── Services/
│   │   ├── AuthService.php                      ← Auth business logic
│   │   └── UserService.php                      ← User business logic
│   │
│   ├── Repositories/
│   │   ├── AuthRepository.php                   ← Auth data access
│   │   ├── UserRepository.php                   ← User data access
│   │   └── Interfaces/
│   │       ├── AuthRepositoryInterface.php
│   │       └── UserRepositoryInterface.php
│   │
│   ├── Models/
│   │   └── User.php                             ← Eloquent model + traits
│   │
│   ├── Mail/
│   │   └── SendUserMail.php                     ← Mailable class
│   │
│   ├── Jobs/
│   │   └── SendUserMailJob.php                  ← Queue job
│   │
│   ├── Providers/
│   │   ├── AppServiceProvider.php               ← Dependency binding
│   │   ├── AuthServiceProvider.php
│   │   ├── BroadcastServiceProvider.php
│   │   ├── EventServiceProvider.php
│   │   └── RouteServiceProvider.php
│   │
│   └── Exceptions/
│       └── Handler.php                          ← Exception handling
│
├── config/
│   ├── app.php                                  ← App configuration
│   ├── auth.php                                 ← Auth guards & providers
│   ├── database.php                             ← Database connections
│   ├── mail.php                                 ← Mail configuration
│   ├── queue.php                                ← Queue configuration
│   └── sanctum.php                              ← Sanctum config
│
├── database/
│   ├── migrations/
│   │   ├── 2014_10_12_000000_create_users_table.php
│   │   ├── 2014_10_12_100000_create_password_reset_tokens_table.php
│   │   ├── 2019_08_19_000000_create_failed_jobs_table.php
│   │   └── 2019_12_14_000001_create_personal_access_tokens_table.php
│   ├── factories/
│   │   └── UserFactory.php
│   └── seeders/
│
├── routes/
│   ├── api.php                                  ← API endpoint definitions
│   ├── web.php                                  ← Web routes (if any)
│   ├── channels.php                             ← Broadcasting channels
│   └── console.php                              ← Console commands
│
├── storage/
│   ├── logs/
│   │   └── laravel.log                          ← Application logs
│   └── app/
│
├── tests/
│   ├── Feature/                                 ← Feature tests
│   ├── Unit/                                    ← Unit tests
│   ├── TestCase.php
│   └── CreatesApplication.php
│
├── artisan                                      ← CLI command
├── composer.json
├── phpunit.xml
└── vite.config.js
```

### Frontend Structure
```
frontend/
├── src/
│   ├── main.jsx                                 ← React entry point
│   ├── App.jsx                                  ← Root component with routes
│   ├── index.css                                ← Global styles
│   │
│   ├── api/
│   │   └── axiosClient.js                       ← Axios configuration
│   │
│   ├── components/
│   │   ├── Header.jsx                           ← Navigation header
│   │   ├── Footer.jsx                           ← Footer
│   │   ├── ProtectedRoutes.jsx                  ← HOC for protected routes
│   │   └── UserModal.jsx                        ← Reusable edit modal
│   │
│   ├── pages/
│   │   ├── Dashboard.jsx                        ← Landing page
│   │   ├── Login.jsx                            ← Login form
│   │   ├── Register.jsx                         ← Registration form
│   │   ├── User.jsx                             ← Admin user list
│   │   └── UserProfile.jsx                      ← User profile display
│   │
│   └── public/                                  ← Static assets
│
├── index.html                                   ← HTML entry point
├── package.json
├── vite.config.js
└── eslint.config.js
```

---

## API Endpoints Reference

### Authentication Endpoints

#### 1. User Registration
```
POST /api/register
Headers: Content-Type: application/json

Request Body:
{
  "name": "John Doe",
  "email": "john@example.com",
  "password": "password123",
  "password_confirmation": "password123"
}

Response (201):
{
  "token": "1|abc123def456...",
  "user": {
    "id": 1,
    "name": "John Doe",
    "email": "john@example.com",
    "phone": null,
    "gender": null,
    "address": null,
    "created_at": "2024-02-17T10:30:00Z"
  },
  "role": "user"
}

Error (422):
{
  "message": "The email has already been taken.",
  "errors": {
    "email": ["The email has already been taken."]
  }
}
```

#### 2. User Login
```
POST /api/login
Headers: Content-Type: application/json

Request Body:
{
  "email": "john@example.com",
  "password": "password123"
}

Response (200):
{
  "token": "2|def456ghi789...",
  "user": {
    "id": 1,
    "name": "John Doe",
    "email": "john@example.com",
    "phone": "1234567890",
    "gender": "male",
    "address": "123 Main St",
    "created_at": "2024-02-17T10:30:00Z"
  },
  "role": "admin" or "user"
}

Error (422):
{
  "message": "The provided credentials are invalid.",
  "errors": {
    "email": ["Invalid credentials."]
  }
}
```

#### 3. User Logout
```
POST /api/logout
Headers: 
  Authorization: Bearer <token>
  Content-Type: application/json

Response (200):
{
  "message": "Logged out successfully"
}

Error (401):
{
  "message": "Unauthenticated."
}
```

### User Management Endpoints (Admin Only)

#### 1. List All Users (Paginated)
```
GET /api/user?page=1&per_page=10
Headers: Authorization: Bearer <admin_token>

Response (200):
{
  "data": [
    {
      "id": 2,
      "name": "Jane Doe",
      "email": "jane@example.com",
      "phone": "9876543210",
      "gender": "female",
      "address": "456 Oak Ave"
    },
    ...
  ],
  "current_page": 1,
  "last_page": 5,
  "per_page": 10,
  "total": 48
}

Error (403):
{
  "message": "Unauthorized role"
}
```

#### 2. Get Single User Details
```
GET /api/user/edit/{id}
Headers: Authorization: Bearer <admin_token>

Response (200):
{
  "id": 2,
  "name": "Jane Doe",
  "email": "jane@example.com",
  "phone": "9876543210",
  "gender": "female",
  "address": "456 Oak Ave",
  "created_at": "2024-02-15T08:00:00Z"
}

Error (404):
{
  "message": "User not found"
}
```

#### 3. Create New User
```
POST /api/user/store
Headers: 
  Authorization: Bearer <admin_token>
  Content-Type: application/json

Request Body:
{
  "name": "Bob Smith",
  "email": "bob@example.com",
  "password": "password123",
  "phone": "5555555555",
  "gender": "male",
  "address": "789 Pine Rd"
}

Response (201):
{
  "id": 3,
  "name": "Bob Smith",
  "email": "bob@example.com",
  ...
}
```

#### 4. Update User
```
POST /api/user/update/{id}
Headers:
  Authorization: Bearer <admin_token>
  Content-Type: application/json

Request Body:
{
  "name": "Jane Smith",
  "phone": "1111111111",
  "gender": "female",
  "address": "999 Elm St"
}

Response (200):
{
  "id": 2,
  "name": "Jane Smith",
  "email": "jane@example.com",
  "phone": "1111111111",
  "gender": "female",
  "address": "999 Elm St"
}
```

#### 5. Delete User
```
DELETE /api/user/delete/{id}
Headers: Authorization: Bearer <admin_token>

Response (200):
{
  "message": "User deleted"
}

Error (404):
{
  "message": "User not found"
}
```

### User Profile Endpoints

#### 1. Get Own Profile
```
GET /api/user/getProfile
Headers: Authorization: Bearer <user_token>

Response (200):
{
  "id": 1,
  "name": "John Doe",
  "email": "john@example.com",
  "phone": "1234567890",
  "gender": "male",
  "address": "123 Main St",
  "created_at": "2024-02-17T10:30:00Z"
}

Error (401):
{
  "message": "Unauthenticated."
}
```

#### 2. Update Own Profile
```
POST /api/user/update/{id}
Headers:
  Authorization: Bearer <user_token>
  Content-Type: application/json

Request Body:
{
  "name": "John Doe Updated",
  "phone": "9999999999",
  "address": "New Address"
}

Note: User can only update own profile (verified via auth()->id())

Response (200):
{
  "id": 1,
  "name": "John Doe Updated",
  "email": "john@example.com",
  "phone": "9999999999",
  "address": "New Address"
}
```

---

## Key Implementation Details

### 1. Service Locator (AppServiceProvider)

```php
// backend/app/Providers/AppServiceProvider.php
public function register(): void
{
    // Interface → Implementation binding
    $this->app->bind(
        UserRepositoryInterface::class,
        UserRepository::class
    );
    $this->app->bind(
        AuthRepositoryInterface::class,
        AuthRepository::class
    );
}
```

**Why?** Allows controllers/services to use interfaces instead of concrete classes. Easy to swap implementations or mock for testing.

### 2. Request Interceptor (Axios)

```javascript
// frontend/src/api/axiosClient.js
axiosClient.interceptors.request.use(config => {
    const token = localStorage.getItem('token');
    if (token) {
        config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
});
```

**Why?** Automatic token injection on every API call. No need to manually add Authorization header in each page.

### 3. Protected Route Component

```jsx
// frontend/src/components/ProtectedRoutes.jsx
const ProtectedRoutes = ({ token, role, allowedRoles, children }) => {
    if (!token) return <Navigate to="/login" />;
    if (allowedRoles && !allowedRoles.includes(role)) {
        return <Navigate to="/" />;
    }
    return children;
};
```

**Why?** Prevents unauthorized access to admin/user-only pages.

### 4. Chunked Query for Large Datasets

```php
// backend/app/Jobs/SendUserMailJob.php
$userService->getUsersForMail()->chunk(100, function ($users) {
    foreach ($users as $user) {
        Mail::to($user->email)->queue(new SendUserMail($user));
    }
});
```

**Why?** Uses cursor to process large user lists without loading entire table into memory at once.

### 5. Pagination with Laravel Paginator

```php
// backend/app/Repositories/UserRepository.php
public function getAll()
{
    // Excludes current user and paginates
    return $this->user
        ->where('id', '<>', auth()->user()->id)
        ->paginate(10);
}
```

Response format:
```json
{
  "data": [...],
  "current_page": 1,
  "last_page": 3,
  "per_page": 10,
  "total": 25
}
```

### 6. Spatie Permission (RBAC)

```php
// backend/app/Services/AuthService.php
$user->assignRole('user'); // During registration

// In routes/api.php
Route::middleware('role:admin')->group(function () {
    // Admin-only routes
});
```

### 7. Queue Job Dispatching

```php
// backend/app/Console/Commands/SendScheduledUserMailCommand.php
public function handle()
{
    SendUserMailJob::dispatch();
}
```

Job will be picked up by `php artisan queue:work` listener.

### 8. Sanctum Token Creation

```php
$token = $user->createToken('auth_token')->plainTextToken;
```

Token format: `id|hash` (example: `1|abc123def456ghi789`)

---

## Development Setup

### Backend Setup

```bash
# Install dependencies
composer install

# Create .env file
cp .env.example .env

# Generate app key
php artisan key:generate

# Run migrations
php artisan migrate

# Create Sanctum keys
php artisan passport:keys (if using Passport instead)

# Optional: Seed demo data
php artisan db:seed

# Start dev server
php artisan serve
```

### Frontend Setup

```bash
# Install dependencies
npm install

# Start dev server
npm run dev

# Build for production
npm run build
```

### Queue & Scheduler Setup

```bash
# Start queue worker (listen for jobs)
php artisan queue:work

# In separate terminal, start scheduler
while true; do
  php artisan schedule:run
  sleep 60
done

# Or add to crontab:
* * * * * cd /path/to/project && php artisan schedule:run >> /dev/null 2>&1
```

---

## Common Patterns Used

### 1. **Service-Repository Pattern**
- Separation of concerns
- Service = Business logic
- Repository = Data access logic
- Interface = Contract for implementations

### 2. **Dependency Injection**
- Constructor injection in classes
- Service container resolves dependencies
- Makes testing easier (mock dependencies)

### 3. **Query Chunking**
- Process large datasets efficiently
- Memory-safe iteration
- Use case: bulk email operations

### 4. **Token-Based Authentication**
- Stateless (no sessions)
- Each request includes Bearer token
- Suitable for SPA + REST API

### 5. **Role-Based Access Control (RBAC)**
- Users assigned roles
- Routes protected by role middleware
- Spatie Permission package handles it

### 6. **Exception Handling**
```php
try {
    // Operation
} catch (\Throwable $e) {
    Log::error('Operation failed', ['error' => $e->getMessage()]);
    throw new \Exception("User-friendly message");
}
```

### 7. **Pagination Pattern**
- Frontend: Track current page in state
- Backend: Return metadata (current_page, last_page, total)
- Use for performance with large datasets

---

## Troubleshooting Guide

### Backend Issues

#### 1. "CORS error when calling API from React"
```php
// backend/config/cors.php
'allowed_origins' => ['http://127.0.0.1:5173', 'http://localhost:5173'],
'allowed_methods' => ['*'],
'allowed_headers' => ['*'],
'exposed_headers' => [],
```

#### 2. "Sanctum token not working"
- Verify token is hashed in database: `PersonalAccessToken` table
- Check token format: should be `id|plaintext`
- Ensure `auth:sanctum` middleware is applied

#### 3. "Role middleware not working"
- Verify user has role: `$user->roles` 
- Check Spatie Permission migrations ran: `php artisan migrate`
- Assign role manually: `$user->assignRole('admin')`

#### 4. "Queue jobs not processing"
- Start queue worker: `php artisan queue:work`
- Check failed jobs: `php artisan queue:failed`
- View failed jobs table for errors

#### 5. "Scheduler not running"
- Verify cron job is set up correctly
- Test manually: `php artisan schedule:run`
- Check logs: `storage/logs/laravel.log`

### Frontend Issues

#### 1. "Token not being sent to API"
```javascript
// Verify localStorage has token
console.log(localStorage.getItem('token'));

// Check Axios interceptor
axiosClient.interceptors.request.use((config) => {
    console.log('Authorization header:', config.headers.Authorization);
    return config;
});
```

#### 2. "Protected routes not redirecting"
- Verify token in localStorage
- Check role matches allowed roles
- Console.log in ProtectedRoute component

#### 3. "Form not submitting"
```javascript
// Check browser console for errors
// Verify form validation
// Check network tab in DevTools
```

#### 4. "Toast notifications not showing"
- Ensure `<ToastContainer />` in App.jsx
- Verify CSS imported: `import "react-toastify/dist/ReactToastify.css"`
- Check `toast.success()` called correctly

#### 5. "Page refresh loses authentication"
```javascript
// Add useEffect in App.jsx
useEffect(() => {
    const token = localStorage.getItem('token');
    if (token) {
        // Verify token is still valid on backend
        // Or reload user data
    }
}, []);
```

### Database Issues

#### 1. "User not found after registration"
- Check migration ran: `php artisan migrate:status`
- Verify users table exists
- Check password hashing: `Hash::make()` used

#### 2. "Foreign key constraint failed"
- Ensure related record exists before referencing
- Check migration dependencies order

---

## Performance Optimization Tips

### Backend
1. **Use pagination** instead of `->get()` on large tables
2. **Use indexing** on frequently queried columns
3. **Query chunking** for bulk operations
4. **Caching** with Redis for frequently accessed data
5. **Eager loading** with `->with()` to avoid N+1 queries

### Frontend
1. **Code splitting** - Lazy load pages with React.lazy()
2. **Memoization** - Use `useMemo`, `useCallback` for expensive operations
3. **Image optimization** - Use WebP, compress images
4. **Minification** - Build production with `npm run build`
5. **Debouncing** - Prevent excessive API calls (search, etc.)

