# flutter_app-terraform-deploy-to-aws-
Flutter App with deployment to AWS


Creating a Flutter mobile app with a NestJS backend that includes user authentication (login) is a great full-stack project. 

#Project Directory
```
project-root/
│
├── flutter-frontend/          # Flutter frontend
│   ├── android/               # Android platform-specific files
│   ├── ios/                   # iOS platform-specific files
│   ├── lib/                   # Main Flutter application code
│   │   ├── auth_service.dart  # Service for API calls to the NestJS backend
│   │   ├── login_screen.dart  # UI for the login screen
│   │   ├── main.dart          # Main entry point for Flutter app
│   │   └── ...                # Additional Flutter code (other screens, widgets)
│   ├── assets/                # Assets like fonts, images
│   ├── pubspec.yaml           # Flutter dependencies and project configuration
│   └── test/                  # Unit and widget tests for Flutter app
│
└── nest-backend/              # NestJS backend
    ├── src/                   # Main application code
    │   ├── auth/              # Auth module for login and registration
    │   │   ├── auth.module.ts
    │   │   ├── auth.service.ts
    │   │   ├── auth.controller.ts
    │   │   └── jwt.strategy.ts
    │   ├── users/             # User module for managing users
    │   │   ├── users.module.ts
    │   │   ├── users.service.ts
    │   │   ├── users.controller.ts
    │   │   └── user.entity.ts
    │   ├── app.module.ts      # Main app module
    │   └── main.ts            # Main entry point for NestJS server
    ├── test/                  # Unit and end-to-end tests
    ├── node_modules/          # Node.js dependencies
    ├── package.json           # Node.js dependencies and project configuration
    ├── tsconfig.json          # TypeScript configuration
    └── .env                   # Environment variables (JWT secret, database credentials)

```

### **Step-by-Step Guide**

---

### **1. Backend (NestJS with JWT Authentication)**

#### a. Install NestJS and Create a Project
First, you'll need to install NestJS and create a new project:

```bash
npm i -g @nestjs/cli
nest new nest-backend
```

#### b. Install Required Dependencies
Inside the NestJS project, install the required dependencies for JWT authentication, bcrypt for password hashing, and managing CORS:

```bash
npm install @nestjs/jwt @nestjs/passport passport passport-jwt bcryptjs
npm install --save-dev @types/bcryptjs
```

#### c. Create User Authentication Logic
1. **Create Modules**:
   Create `Auth`, `Users`, and `JWT` strategies:
   ```bash
   nest generate module auth
   nest generate module users
   nest generate service users
   nest generate controller users
   nest generate service auth
   nest generate controller auth
   ```

2. **Create User Entity**:
   Create a simple `User` model (assuming you're using TypeORM or another ORM).

   Example `User` model:
   ```ts
   import { Entity, Column, PrimaryGeneratedColumn } from 'typeorm';

   @Entity()
   export class User {
     @PrimaryGeneratedColumn()
     id: number;

     @Column({ unique: true })
     email: string;

     @Column()
     password: string;
   }
   ```

3. **AuthService and JWT Strategy**:
   Implement JWT-based authentication in `auth.service.ts`.

   Example `auth.service.ts`:
   ```ts
   import { Injectable, UnauthorizedException } from '@nestjs/common';
   import { JwtService } from '@nestjs/jwt';
   import * as bcrypt from 'bcryptjs';
   import { UsersService } from '../users/users.service';

   @Injectable()
   export class AuthService {
     constructor(private usersService: UsersService, private jwtService: JwtService) {}

     async validateUser(email: string, pass: string): Promise<any> {
       const user = await this.usersService.findByEmail(email);
       if (user && bcrypt.compareSync(pass, user.password)) {
         const { password, ...result } = user;
         return result;
       }
       return null;
     }

     async login(user: any) {
       const payload = { email: user.email, sub: user.id };
       return {
         access_token: this.jwtService.sign(payload),
       };
     }

     async register(email: string, password: string) {
       const hashedPassword = bcrypt.hashSync(password, 10);
       return this.usersService.create(email, hashedPassword);
     }
   }
   ```

4. **JWT Module and Middleware**:
   Add JWT logic inside `auth.module.ts`.

   Example `auth.module.ts`:
   ```ts
   import { Module } from '@nestjs/common';
   import { JwtModule } from '@nestjs/jwt';
   import { PassportModule } from '@nestjs/passport';
   import { AuthService } from './auth.service';
   import { UsersModule } from '../users/users.module';
   import { JwtStrategy } from './jwt.strategy';

   @Module({
     imports: [
       UsersModule,
       PassportModule,
       JwtModule.register({
         secret: 'your-secret-key', // use environment variables in production
         signOptions: { expiresIn: '60m' },
       }),
     ],
     providers: [AuthService, JwtStrategy],
     exports: [AuthService],
   })
   export class AuthModule {}
   ```

5. **Routes for Registration & Login**:
   Define routes in `auth.controller.ts`.

   Example `auth.controller.ts`:
   ```ts
   import { Controller, Post, Body } from '@nestjs/common';
   import { AuthService } from './auth.service';

   @Controller('auth')
   export class AuthController {
     constructor(private readonly authService: AuthService) {}

     @Post('login')
     async login(@Body() body) {
       return this.authService.login(body);
     }

     @Post('register')
     async register(@Body() body) {
       return this.authService.register(body.email, body.password);
     }
   }
   ```

#### d. Enable CORS and Set Up API Endpoints
In `main.ts`, enable CORS to allow the Flutter app to communicate with your NestJS backend:

```ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.enableCors();  // Enable CORS for all routes
  await app.listen(3000);
}
bootstrap();
```

#### e. Run the NestJS Backend

```bash
npm run start
```

---

### **2. Frontend (Flutter App)**

#### a. Set Up a New Flutter Project

```bash
flutter create flutter_frontend
cd flutter_frontend
```

#### b. Add Dependencies to `pubspec.yaml`

Add these dependencies for HTTP requests and form management:

```yaml
dependencies:
  flutter:
    sdk: flutter
  http: ^0.14.0
  provider: ^6.0.0
```

Run:

```bash
flutter pub get
```

#### c. Create Authentication Service in Flutter
Use the `http` package to connect to your NestJS backend. Create a service to handle user login and registration.

Example `auth_service.dart`:

```dart
import 'dart:convert';
import 'package:http/http.dart' as http;

class AuthService {
  final String apiUrl = 'http://localhost:3000'; // Your NestJS API

  Future<String?> login(String email, String password) async {
    final response = await http.post(
      Uri.parse('$apiUrl/auth/login'),
      headers: {'Content-Type': 'application/json'},
      body: json.encode({'email': email, 'password': password}),
    );

    if (response.statusCode == 200) {
      final data = json.decode(response.body);
      return data['access_token']; // JWT Token
    } else {
      return null;
    }
  }

  Future<bool> register(String email, String password) async {
    final response = await http.post(
      Uri.parse('$apiUrl/auth/register'),
      headers: {'Content-Type': 'application/json'},
      body: json.encode({'email': email, 'password': password}),
    );

    return response.statusCode == 201;
  }
}
```

#### d. Create Login Screen

Create a login screen with two input fields for email and password.

Example `login_screen.dart`:

```dart
import 'package:flutter/material.dart';
import 'auth_service.dart';

class LoginScreen extends StatefulWidget {
  @override
  _LoginScreenState createState() => _LoginScreenState();
}

class _LoginScreenState extends State<LoginScreen> {
  final _emailController = TextEditingController();
  final _passwordController = TextEditingController();
  final AuthService _authService = AuthService();

  _login() async {
    final token = await _authService.login(
      _emailController.text,
      _passwordController.text,
    );
    
    if (token != null) {
      print('Login successful, JWT Token: $token');
      // Navigate to a new screen or save token
    } else {
      print('Login failed');
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Login')),
      body: Padding(
        padding: const EdgeInsets.all(16.0),
        child: Column(
          children: [
            TextField(
              controller: _emailController,
              decoration: InputDecoration(labelText: 'Email'),
            ),
            TextField(
              controller: _passwordController,
              decoration: InputDecoration(labelText: 'Password'),
              obscureText: true,
            ),
            SizedBox(height: 20),
            ElevatedButton(
              onPressed: _login,
              child: Text('Login'),
            ),
          ],
        ),
      ),
    );
  }
}
```

#### e. Create Main App

Update your `main.dart` to display the login screen.

Example `main.dart`:

```dart
import 'package:flutter/material.dart';
import 'login_screen.dart';

void main() {
  runApp(MyApp());
}

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Auth App',
      theme: ThemeData(primarySwatch: Colors.blue),
      home: LoginScreen(),
    );
  }
}
```

#### f. Run the Flutter App

Run the Flutter app on an emulator or connected device:

```bash
flutter run
```

---

### **Summary**

- **Backend**: NestJS provides API endpoints for user registration and login, secured with JWT.
- **Frontend**: Flutter handles the UI for login and communicates with the NestJS backend for authentication.

You now have a basic login system where users can log in using a Flutter frontend and NestJS backend! Let me know if you need any additional details or refinements!
