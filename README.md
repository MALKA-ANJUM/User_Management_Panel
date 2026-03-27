# Interview Project - Full-Stack User Management System

A comprehensive full-stack application built with Laravel 11 (backend) and React 19 (frontend), demonstrating enterprise-level architecture patterns including service layers, repository patterns, role-based access control (RBAC), queue-based job processing, and automated task scheduling.

## 🚀 Features

- **User Authentication & Authorization**: Token-based authentication using Laravel Sanctum with role-based access control (Admin/User roles)
- **User Management**: Admins can perform CRUD operations on users; users can manage their own profiles
- **Background Job Processing**: Asynchronous email notifications via Laravel queues
- **Task Scheduling**: Automated commands for email sending and daily cleanup
- **Responsive UI**: Modern React frontend with Bootstrap styling and toast notifications
- **API-Driven Architecture**: RESTful API with protected routes and middleware
- **Scalable Design**: Service/repository layers, dependency injection, and clean separation of concerns

## 🛠️ Tech Stack

### Backend
- **Laravel 11**: PHP framework for robust backend development
- **Laravel Sanctum**: API token authentication
- **Spatie Laravel Permission**: Role-based access control
- **Eloquent ORM**: Database interactions
- **Queue System**: Asynchronous job processing
- **Task Scheduler**: Cron-like automation

### Frontend
- **React 19**: Modern UI library
- **React Router 7**: Client-side routing with protected routes
- **Axios 1.13**: HTTP client for API calls
- **Bootstrap 5.3**: Responsive CSS framework
- **React-Toastify 11**: Toast notifications
- **Vite 5**: Fast build tool and dev server

## 📋 Prerequisites

- PHP 8.1 or higher
- Composer
- Node.js 18+ and npm
- MySQL or compatible database
- Laravel requirements (check `backend/composer.json`)

## 🔧 Installation

### Backend Setup

1. **Navigate to backend directory**:
   ```bash
   cd backend
   ```

2. **Install PHP dependencies**:
   ```bash
   composer install
   ```

3. **Environment configuration**:
   ```bash
   cp .env.example .env
   ```
   Update `.env` with your database credentials and other settings:
   ```env
   DB_CONNECTION=mysql
   DB_HOST=127.0.0.1
   DB_PORT=3306
   DB_DATABASE=interview_project
   DB_USERNAME=your_username
   DB_PASSWORD=your_password
   ```

4. **Generate application key**:
   ```bash
   php artisan key:generate
   ```

5. **Run migrations and seeders**:
   ```bash
   php artisan migrate
   php artisan db:seed
   ```

6. **Install permissions**:
   ```bash
   php artisan permission:create-role admin
   php artisan permission:create-role user
   ```

7. **Start the Laravel server**:
   ```bash
   php artisan serve
   ```
   Backend will run on `http://127.0.0.1:8000`

### Frontend Setup

1. **Navigate to frontend directory**:
   ```bash
   cd frontend
   ```

2. **Install Node dependencies**:
   ```bash
   npm install
   ```

3. **Start the development server**:
   ```bash
   npm run dev
   ```
   Frontend will run on `http://localhost:5173` (Vite default)

## 🚀 Usage

### Accessing the Application

1. Open your browser and go to `http://localhost:5173`
2. Register a new account or login with existing credentials
3. Based on your role:
   - **Admin**: Access user management at `/user` for CRUD operations
   - **User**: Access profile management at `/user/profile`

### API Endpoints

#### Public Endpoints
- `POST /api/register` - User registration
- `POST /api/login` - User login

#### Protected Endpoints (require authentication)
- `POST /api/logout` - User logout

#### Admin-Only Endpoints
- `GET /api/user` - List all users (paginated)
- `POST /api/user/store` - Create new user
- `GET /api/user/edit/{id}` - Get user details for editing
- `POST /api/user/update/{id}` - Update user
- `DELETE /api/user/delete/{id}` - Delete user

#### User Endpoints
- `GET /api/user/getProfile` - Get current user profile

### Running Background Jobs

1. **Start queue worker**:
   ```bash
   php artisan queue:work
   ```

2. **Run scheduled tasks** (add to crontab):
   ```bash
   * * * * * cd /path-to-project/backend && php artisan schedule:run >> /dev/null 2>&1
   ```

3. **Manual commands**:
   ```bash
   # Send scheduled emails
   php artisan users:send-mail

   # Daily cleanup
   php artisan cleanup:daily
   ```

## 🏗️ Project Structure

```
interview-project/
├── backend/                 # Laravel application
│   ├── app/
│   │   ├── Http/Controllers/
│   │   ├── Services/
│   │   ├── Repositories/
│   │   ├── Jobs/
│   │   └── Models/
│   ├── database/migrations/
│   ├── routes/api.php
│   └── composer.json
├── frontend/                # React application
│   ├── src/
│   │   ├── components/
│   │   ├── pages/
│   │   ├── api/
│   │   └── App.jsx
│   ├── public/
│   └── package.json
├── PROJECT_DOCUMENTATION.md # Detailed architecture docs
└── README.md               # This file
```

## 🧪 Testing

### Backend Tests
```bash
cd backend
php artisan test
```

### Frontend Tests
```bash
cd frontend
npm test
```

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

## 📝 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 📞 Support

For questions or issues, please open an issue in the repository or contact the development team.

---

**Note**: This project is designed for educational and demonstration purposes, showcasing modern full-stack development practices with Laravel and React.
