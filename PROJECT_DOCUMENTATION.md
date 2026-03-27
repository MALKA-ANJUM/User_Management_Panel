# Interview Project - Architecture Documentation

## Table of Contents
1. [Project Overview](#project-overview)
2. [Backend Architecture](#backend-architecture)
3. [Frontend Architecture](#frontend-architecture)
4. [Key Technologies & Libraries](#key-technologies--libraries)
5. [Quick Summary (25-Second Read)](#quick-summary-25-second-read)

---

## Project Overview

This is a **full-stack Laravel + React application** designed with high-level enterprise architecture patterns. The project demonstrates best practices including service layers, repository patterns, queue-based job processing, scheduled tasks, role-based access control (RBAC), and modern frontend authentication with token-based security.

**Purpose**: A user management system with admin capabilities, allowing admins to manage users and users to manage their own profiles, with automated email notifications via background jobs.

---

## Backend Architecture

### 1. High-Level Architecture Pattern

The backend follows a **layered architecture** with clear separation of concerns:

```
HTTP Request
    ↓
Routes (api.php)
    ↓
Controllers (Validation & Request Handling)
    ↓
Services (Business Logic)
    ↓
Repositories (Data Access Layer)
    ↓
Database Models (Eloquent ORM)
    ↓
Database
```

### 2. Service Layer

**Location**: `backend/app/Services/`

The service layer contains all business logic and is not directly accessed by controllers. Services handle:
- Data validation and processing
- Business rule enforcement
- Logging and error handling
- Dependency injection of repositories

#### AuthService.php
Handles user authentication operations:
- **register(array $data)**: Creates new user, assigns 'user' role, generates token
- **login(array $data)**: Validates credentials, generates Sanctum token
- **logout($user)**: Revokes all tokens for a user

```php
// Example: AuthService uses repository through interface
public function __construct(AuthRepositoryInterface $authRepository)
{
    $this->authRepository = $authRepository;
}

public function register(array $data)
{
    $data['password'] = Hash::make($data['password']);
    $user = $this->authRepository->create($data);
    $user->assignRole('user');
    $token = $user->createToken('auth_token')->plainTextToken;
    return ['token' => $token, 'user' => $user, 'role' => 'user'];
}
```

#### UserService.php
Manages user-related operations:
- **getAllUsers()**: Retrieves paginated users (admin only)
- **getUser($id)**: Fetches individual user details
- **createUser(array $data)**: Creates new user with validation
- **updateUser($id, array $data)**: Updates user information
- **deleteUser($id)**: Deletes user record
- **getUsersForMail()**: Retrieves active users for bulk email operations
- **getProfile()**: Gets authenticated user's profile

All methods include comprehensive logging and error handling.

### 3. Repository Pattern & Interface Layer

**Location**: `backend/app/Repositories/` & `backend/app/Repositories/Interfaces/`

The repository pattern abstracts database operations and enforces dependency inversion principle.

#### Interfaces
Define contracts that implementations must follow:

**AuthRepositoryInterface**:
```php
interface AuthRepositoryInterface
{
    public function create(array $data);
    public function findByEmail(string $email);
}
```

**UserRepositoryInterface**:
```php
interface UserRepositoryInterface
{
    public function getAll();
    public function insert(array $data);
    public function edit($id);
    public function update($id, array $data);
    public function delete($id);
    public function getAllActiveUsers();
    public function getProfile();
}
```

#### Repository Implementations

**AuthRepository**: Implements AuthRepositoryInterface
- Handles user creation and email lookups
- Works directly with User model

**UserRepository**: Implements UserRepositoryInterface
- Excludes current user from listings
- Implements pagination (10 per page)
- Cursor-based iteration for bulk operations
- Role-aware profile retrieval

### 4. Dependency Injection & Service Registration

**Location**: `backend/app/Providers/AppServiceProvider.php`

```php
public function register(): void
{
    $this->app->bind(UserRepositoryInterface::class, UserRepository::class);
    $this->app->bind(AuthRepositoryInterface::class, AuthRepository::class);
}
```

This ensures:
- Loose coupling between services and repositories
- Easy testing through interface mocking
- Centralized dependency configuration

### 5. Controllers

**Location**: `backend/app/Http/Controllers/`

Controllers handle HTTP requests, validation, and response formatting. They delegate business logic to services.

#### AuthController
```php
public function register(Request $request)
{
    $validated = $request->validate([
        'name' => 'required|string|max:255',
        'email' => 'required|email|unique:users',
        'password' => 'required|min:6|confirmed',
    ]);
    return response()->json($this->authService->register($validated));
}

public function login(Request $request)
{
    return response()->json($this->authService->login($request->only('email','password')));
}

public function logout(Request $request)
{
    $this->authService->logout($request->user());
    return response()->json(['message' => 'Logged out successfully']);
}
```

#### UserController
```php
public function list()
{
    return response()->json($this->userService->getAllUsers());
}

public function store(Request $request)
{
    return response()->json($this->userService->createUser($request->all()));
}

public function update(Request $request, $id)
{
    return response()->json($this->userService->updateUser($id, $request->all()));
}

public function delete($id)
{
    $this->userService->deleteUser($id);
    return response()->json(['message' => 'User deleted']);
}

public function getProfile()
{
    return response()->json($this->userService->getProfile());
}
```

### 6. API Routes & Middleware

**Location**: `backend/routes/api.php`

**Public Routes**:
- `POST /register` - User registration
- `POST /login` - User login

**Protected Routes** (require `auth:sanctum` middleware):
- `POST /logout` - User logout

**Role-Based Routes**:
```php
// Admin-only endpoints
Route::middleware('role:admin')->group(function () {
    Route::get('/user', [UserController::class, 'list']);
    Route::post('/user/store', [UserController::class, 'store']);
    Route::get('/user/edit/{id}', [UserController::class, 'edit']);
    Route::delete('/user/delete/{id}', [UserController::class, 'delete']);
});

// User-only endpoints
Route::middleware('role:user')->group(function () {
    Route::get('/user/getProfile', [UserController::class, 'getProfile']);
});
```

### 7. Database & Models

**Model**: `backend/app/Models/User.php`

Uses Laravel Eloquent with traits for extended functionality:
- **HasApiTokens**: Enables Sanctum token authentication
- **HasRoles**: Spatie permission package for RBAC
- **Notifiable**: Enables mail and notification features

```php
protected $fillable = [
    'name', 'email', 'phone', 'gender', 'address', 'password',
];

protected $casts = [
    'email_verified_at' => 'datetime',
    'password' => 'hashed',
];
```

### 8. Jobs & Queue System

**Location**: `backend/app/Jobs/SendUserMailJob.php`

Implements asynchronous email sending:

```php
class SendUserMailJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, SerializesModels;

    public function handle(UserService $userService)
    {
        // Chunk large user lists to prevent memory overflow
        $userService->getUsersForMail()->chunk(100, function ($users) {
            foreach ($users as $user) {
                Mail::to($user->email)->queue(new SendUserMail($user));
            }
        });
    }
}
```

**Features**:
- Implements `ShouldQueue` for background processing
- Uses cursor-based chunking for memory efficiency
- Dependency injection of UserService
- Can be dispatched from commands or scheduled tasks

### 9. Mailing System

**Location**: `backend/app/Mail/SendUserMail.php`

```php
class SendUserMail extends Mailable
{
    public $user;

    public function build()
    {
        return $this->subject('Scheduled Mail')
                    ->view('emails.user-mail');
    }
}
```

Mail queuing ensures non-blocking email operations.

### 10. Scheduling & Commands

**Location**: `backend/app/Console/Kernel.php`

```php
protected function schedule(Schedule $schedule): void
{
    // Run every 5 minutes, preventing overlaps
    $schedule->command('users:send-mail')
         ->everyFiveMinutes()
         ->withoutOverlapping()
         ->runInBackground();

    // Run daily at 2:00 AM
    $schedule->command('cleanup:daily')->dailyAt('02:00');
}
```

#### Custom Commands

**SendScheduledUserMailCommand** (`users:send-mail`)
```php
protected $signature = 'users:send-mail';

public function handle()
{
    SendUserMailJob::dispatch();
    $this->info('Mail job dispatched successfully');
}
```

**DailyCleanup** (`cleanup:daily`)
```php
protected $signature = 'cleanup:daily';

public function handle()
{
    // Remove expired Sanctum tokens
    $expiredTokens = PersonalAccessToken::where('expires_at', '<', now())->delete();
    
    // Delete old logs (>30 days)
    // Clear temporary data
}
```

**Setup**: Add Laravel scheduler to crontab:
```bash
* * * * * cd /path && php artisan schedule:run >> /dev/null 2>&1
```

### 11. Authentication & Authorization

**Guard**: Laravel Sanctum (token-based API authentication)

**Roles & Permissions**: Spatie Permission package

```php
// User roles created during registration
$user->assignRole('user'); // Regular users
// Admins assigned separately with permissions
```

**Role-Based Access Control**:
- **admin**: Can list, create, edit, delete users
- **user**: Can view own profile, update own data

---

## Frontend Architecture

### 1. React Setup

**Build Tool**: Vite (fast development server, optimized builds)

**Location**: `frontend/`

**Entry Point**: `src/main.jsx`

```jsx
import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import App from './App.jsx'
import 'bootstrap/dist/css/bootstrap.min.css'

createRoot(document.getElementById('root')).render(
    <StrictMode>
        <App />
    </StrictMode>,
)
```

### 2. Routing Architecture

**Location**: `src/App.jsx`

Implements React Router v7 with nested route definitions, protected routes, and role-based access control.

```jsx
function App() {
    const [token, setToken] = useState(localStorage.getItem('token'));
    const [role, setRole] = useState(localStorage.getItem('role'));

    useEffect(() => {
        if (token) localStorage.setItem('token', token);
        else localStorage.removeItem('token');
    }, [token, role]);

    return (
        <BrowserRouter>
            <Routes>
                <Route path="/" element={<Dashboard />} />
                <Route path="/login" element={<Login setToken={setToken} setRole={setRole} />} />
                <Route path="/register" element={<Register />} />
                
                {/* Protected Routes */}
                <Route path="/user" element={
                    <ProtectedRoute token={token} role={role} allowedRoles={['admin']}>
                        <User />
                    </ProtectedRoute>
                } />
                
                <Route path="/user/profile" element={
                    <ProtectedRoute token={token} role={role} allowedRoles={['user','admin']}>
                        <UserProfile />
                    </ProtectedRoute>
                } />
                
                <Route path="*" element={<Navigate to="/" />} />
            </Routes>
        </BrowserRouter>
    );
}
```

### 3. Protected Routes Component

**Location**: `src/components/ProtectedRoutes.jsx`

```jsx
const ProtectedRoutes = ({ token, role, allowedRoles, children }) => {
    if (!token) {
        return <Navigate to="/login" />;
    }

    if (allowedRoles && !allowedRoles.includes(role)) {
        return <Navigate to="/" />;
    }

    return children;
};
```

**Features**:
- Checks for token existence (authentication)
- Validates user role against allowed roles (authorization)
- Redirects to login or home if unauthorized

### 4. API Client Setup

**Location**: `src/api/axiosClient.js`

Central Axios configuration for all API calls:

```javascript
const axiosClient = axios.create({
    baseURL: 'http://127.0.0.1:8000/api',
});

// Request interceptor: Attach Bearer token to all requests
axiosClient.interceptors.request.use(config => {
    const token = localStorage.getItem('token');
    if (token) {
        config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
});

export default axiosClient;
```

**Features**:
- Centralized base URL configuration
- Automatic JWT token injection
- Single point for API configuration changes

### 5. Authentication Pages

#### Login Page (`src/pages/Login.jsx`)

```jsx
const handleSubmit = async (e) => {
    e.preventDefault();
    try {
        const { data } = await axiosClient.post('/login', { email, password });
        setToken(data.token);
        setRole(data.role);
        localStorage.setItem('token', data.token);
        localStorage.setItem('role', data.role);
        toast.success('Login successful!');
        
        // Role-based redirect
        if (data.role === 'admin') {
            navigate('/user');
        } else {
            navigate('/user/profile');
        }
    } catch (err) {
        toast.error(err.response?.data?.message || 'Invalid credentials');
    }
};
```

**Features**:
- Email and password validation
- Token and role storage in localStorage
- Role-based navigation after login
- Toast notifications for user feedback

#### Register Page (`src/pages/Register.jsx`)

```jsx
const handleSubmit = async (e) => {
    e.preventDefault();
    try {
        const { data } = await axiosClient.post("/register", form);
        localStorage.setItem("token", data.token);
        localStorage.setItem("role", data.role);
        toast.success("Registration successful!");
        navigate("/user/profile");
    } catch (err) {
        toast.error(err.response?.data?.message || "Registration failed");
    }
};
```

**Fields**: name, email, password, password_confirmation

### 6. User Management Pages

#### Admin User Management (`src/pages/User.jsx`)

```jsx
const fetchUser = async (p = 1) => {
    const res = await axiosClient.get('/user', { params: { page: p, per_page: 10 }});
    const payload = res.data;
    setUsers(payload.data || []);
    setPage(payload.current_page || p);
    setLastPage(payload.last_page || 1);
};

const handleDelete = async (id) => {
    if (!confirm('Delete this user?')) return;
    await axiosClient.delete(`/user/delete/${id}`);
    toast.success("User deleted successfully");
    fetchUser(page);
};

const handleSave = async (formData) => {
    await axiosClient.post(`/user/update/${editUser.id}`, formData);
    toast.success("User updated successfully");
    fetchUser(page);
};
```

**Features**:
- Paginated user listing (10 per page)
- Edit functionality with modal
- Delete with confirmation
- Real-time table updates
- Toast notifications for actions

**Table Display**:
- Name, Phone, Email, Address
- Edit and Delete action buttons

#### User Profile Page (`src/pages/UserProfile.jsx`)

```jsx
useEffect(() => {
    axiosClient.get("/user/getProfile")
        .then((res) => setUser(res.data))
        .catch((err) => console.error("Profile error:", err));
}, []);
```

**Display Format**:
- Styled card layout
- Profile information: name, email, phone, address, gender
- Edit profile capability via modal
- Success/error toast notifications

### 7. Reusable Components

#### Header Component (`src/components/Header.jsx`)

```jsx
export default function Header() {
    const token = localStorage.getItem('token');

    return (
        <header className="header bg-success text-white py-3">
            <div className="container d-flex justify-content-between">
                <h4 className="fw-bold" onClick={() => navigate('/')}>
                    Project
                </h4>
                <nav>
                    {token ? (
                        <button className="btn btn-light" onClick={handleLogout}>
                            Logout
                        </button>
                    ) : (
                        <>
                            <button className="btn btn-outline-light" onClick={() => navigate('/login')}>
                                Login
                            </button>
                            <button className="btn btn-light" onClick={() => navigate('/register')}>
                                Register
                            </button>
                        </>
                    )}
                </nav>
            </div>
        </header>
    );
}
```

**Features**:
- Conditional rendering based on authentication state
- Navigation links
- Logout functionality
- Bootstrap styling

#### User Modal Component (`src/components/UserModal.jsx`)

Modal dialog for editing user information, used in both User and UserProfile pages.

#### Footer Component (`src/components/Footer.jsx`)

Footer displayed on all pages.

### 8. State Management

**Token & Role Storage**:
- Stored in browser `localStorage` for persistence
- Synced to React state for reactive UI updates
- Token automatically added to all API requests via Axios interceptor

### 9. Notification System

**Toast Notifications** (`react-toastify`):

```jsx
import { ToastContainer, toast } from "react-toastify";
import "react-toastify/dist/ReactToastify.css";

// In App.jsx
<ToastContainer />

// Usage
toast.success("Operation successful!");
toast.error("Something went wrong!");
```

**Types Used**:
- Success notifications after CRUD operations
- Error notifications for failed requests
- Confirmation dialogs before destructive actions

### 10. Styling

**Framework**: Bootstrap 5.3

```jsx
import 'bootstrap/dist/css/bootstrap.min.css';
```

Provides:
- Responsive grid system
- Pre-built components (buttons, cards, tables, modals)
- Utility classes (spacing, sizing, colors)
- Dark mode support

---

## Key Technologies & Libraries

### Backend
| Technology | Purpose |
|---|---|
| **Laravel 11** | Web framework |
| **Sanctum** | API token authentication |
| **Spatie Permission** | Role-based access control |
| **Eloquent ORM** | Database interactions |
| **Queue System** | Asynchronous job processing |
| **Task Scheduling** | Cron-like task automation |
| **Mail** | Email sending capabilities |

### Frontend
| Technology | Purpose |
|---|---|
| **React 19** | UI library |
| **React Router 7** | Client-side routing |
| **Axios 1.13** | HTTP client for API calls |
| **Bootstrap 5.3** | CSS framework |
| **React-Toastify 11** | Toast notifications |
| **Vite 5** | Build tool & dev server |

---

## Quick Summary (25-Second Read)

### 🎯 Project Type
Full-stack **Laravel + React** user management system with role-based access control, background jobs, and task scheduling.

### 🏗️ Backend Architecture
- **Service Layer**: Encapsulates business logic
- **Repository Pattern**: Abstract database access via interfaces
- **Controllers**: Request validation and response formatting
- **Dependency Injection**: Loose coupling through interfaces
- **Authentication**: Sanctum tokens with Spatie roles/permissions

### ⚙️ Backend Features
- User registration/login with token auth
- Admin manages users (CRUD)
- Queue-based email sending every 5 minutes
- Daily cleanup command (expired tokens, old logs)
- Comprehensive logging
- Error handling & validation

### 🎨 Frontend Architecture
- **React Router**: Protected routes with role-based access
- **Axios Client**: Centralized API calls with automatic token injection
- **State Management**: localStorage for token/role
- **Components**: Reusable Header, Footer, Protected Routes, Modals
- **Notifications**: Toast messages for all operations

### ✨ Key Features
- **Authentication**: Token-based with localStorage persistence
- **Authorization**: Admin vs User role restrictions
- **Pagination**: User listings (10 per page)
- **Validation**: Server-side and form validation
- **UX**: Bootstrap styling, toast notifications, password confirmation
- **Background Jobs**: Mail sending via queue
- **Task Automation**: Scheduler for recurring tasks

### 📊 Data Flow
1. Frontend sends request with Bearer token
2. Axios interceptor attaches token
3. API validates token & role via Sanctum + Spatie middleware
4. Controller delegates to Service
5. Service calls Repository for data
6. Repository queries Eloquent Model
7. Response flows back through layers
8. Frontend updates UI with toast notification

