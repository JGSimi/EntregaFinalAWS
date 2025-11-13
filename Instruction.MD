# 🚀 GUIA COMPLETO - PROJETO SPRING SECURITY + ANGULAR + AWS

## 📋 ÍNDICE
1. [Setup Inicial](#setup-inicial)
2. [Back-End (Spring Boot + Security)](#back-end)
3. [Front-End (Angular)](#front-end)
4. [Deploy na AWS](#deploy-aws)
5. [Testes Finais](#testes-finais)

---

## 🔧 SETUP INICIAL

### Ferramentas Necessárias
- Java 17+
- Maven ou Gradle
- Node.js 18+
- Angular CLI (`npm install -g @angular/cli`)
- Git
- Conta AWS (AWS Academy ou conta pessoal)
- Postman (para testes)

---

## 🔐 BACK-END (SPRING BOOT + SECURITY)

### PASSO 1: Criar Projeto Spring Boot

Acesse [start.spring.io](https://start.spring.io) e adicione:
- Spring Web
- Spring Security
- Spring Data JPA
- PostgreSQL Driver (ou MySQL)
- Lombok
- Validation

### PASSO 2: Estrutura de Pastas

```
src/main/java/com/seuprojeto/
├── config/
│   ├── SecurityConfig.java
│   └── JwtConfig.java
├── controller/
│   ├── AuthController.java
│   └── UserController.java
├── model/
│   ├── User.java
│   └── Role.java
├── repository/
│   └── UserRepository.java
├── service/
│   ├── AuthService.java
│   └── JwtService.java
├── dto/
│   ├── LoginRequest.java
│   ├── LoginResponse.java
│   └── RegisterRequest.java
├── security/
│   └── JwtAuthenticationFilter.java
└── Application.java
```

### PASSO 3: Dependências (pom.xml)

Adicione ao seu `pom.xml`:

```xml
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.3</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.3</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.3</version>
    <scope>runtime</scope>
</dependency>
```

### PASSO 4: Arquivo .env (application.properties)

Crie `src/main/resources/application.properties`:

```properties
spring.application.name=seu-projeto

spring.datasource.url=jdbc:postgresql://localhost:5432/seudb
spring.datasource.username=postgres
spring.datasource.password=senha
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true

jwt.secret=${JWT_SECRET:chave-super-secreta-minimo-256-bits-para-HS256-algoritmo}
jwt.expiration=86400000

server.port=8080
```

**IMPORTANTE**: No deploy AWS, você vai criar uma variável de ambiente `JWT_SECRET`.

### PASSO 5: Model User.java

```java
package com.seuprojeto.model;

import jakarta.persistence.*;
import lombok.Data;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.userdetails.UserDetails;

import java.util.Collection;
import java.util.List;

@Entity
@Table(name = "users")
@Data
public class User implements UserDetails {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(unique = true, nullable = false)
    private String username;
    
    @Column(nullable = false)
    private String password;
    
    @Column(nullable = false)
    private String email;
    
    @Enumerated(EnumType.STRING)
    private Role role;
    
    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return List.of(new SimpleGrantedAuthority("ROLE_" + role.name()));
    }
    
    @Override
    public boolean isAccountNonExpired() {
        return true;
    }
    
    @Override
    public boolean isAccountNonLocked() {
        return true;
    }
    
    @Override
    public boolean isCredentialsNonExpired() {
        return true;
    }
    
    @Override
    public boolean isEnabled() {
        return true;
    }
}
```

### PASSO 6: Enum Role.java

```java
package com.seuprojeto.model;

public enum Role {
    USER,
    ADMIN
}
```

### PASSO 7: Repository

```java
package com.seuprojeto.repository;

import com.seuprojeto.model.User;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.Optional;

public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByUsername(String username);
    boolean existsByUsername(String username);
}
```

### PASSO 8: DTOs

**LoginRequest.java**
```java
package com.seuprojeto.dto;

import lombok.Data;

@Data
public class LoginRequest {
    private String username;
    private String password;
}
```

**LoginResponse.java**
```java
package com.seuprojeto.dto;

import lombok.AllArgsConstructor;
import lombok.Data;

@Data
@AllArgsConstructor
public class LoginResponse {
    private String token;
    private String username;
    private String role;
}
```

**RegisterRequest.java**
```java
package com.seuprojeto.dto;

import com.seuprojeto.model.Role;
import lombok.Data;

@Data
public class RegisterRequest {
    private String username;
    private String password;
    private String email;
    private Role role;
}
```

### PASSO 9: JwtService.java

```java
package com.seuprojeto.service;

import io.jsonwebtoken.Claims;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.SignatureAlgorithm;
import io.jsonwebtoken.security.Keys;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.stereotype.Service;

import javax.crypto.SecretKey;
import java.util.Date;
import java.util.HashMap;
import java.util.Map;
import java.util.function.Function;

@Service
public class JwtService {
    
    @Value("${jwt.secret}")
    private String secret;
    
    @Value("${jwt.expiration}")
    private Long expiration;
    
    private SecretKey getSigningKey() {
        return Keys.hmacShaKeyFor(secret.getBytes());
    }
    
    public String extractUsername(String token) {
        return extractClaim(token, Claims::getSubject);
    }
    
    public Date extractExpiration(String token) {
        return extractClaim(token, Claims::getExpiration);
    }
    
    public <T> T extractClaim(String token, Function<Claims, T> claimsResolver) {
        final Claims claims = extractAllClaims(token);
        return claimsResolver.apply(claims);
    }
    
    private Claims extractAllClaims(String token) {
        return Jwts.parserBuilder()
                .setSigningKey(getSigningKey())
                .build()
                .parseClaimsJws(token)
                .getBody();
    }
    
    private Boolean isTokenExpired(String token) {
        return extractExpiration(token).before(new Date());
    }
    
    public String generateToken(UserDetails userDetails) {
        Map<String, Object> claims = new HashMap<>();
        claims.put("role", userDetails.getAuthorities().iterator().next().getAuthority());
        return createToken(claims, userDetails.getUsername());
    }
    
    private String createToken(Map<String, Object> claims, String subject) {
        return Jwts.builder()
                .setClaims(claims)
                .setSubject(subject)
                .setIssuedAt(new Date(System.currentTimeMillis()))
                .setExpiration(new Date(System.currentTimeMillis() + expiration))
                .signWith(getSigningKey(), SignatureAlgorithm.HS256)
                .compact();
    }
    
    public Boolean validateToken(String token, UserDetails userDetails) {
        final String username = extractUsername(token);
        return (username.equals(userDetails.getUsername()) && !isTokenExpired(token));
    }
}
```

### PASSO 10: AuthService.java

```java
package com.seuprojeto.service;

import com.seuprojeto.dto.LoginRequest;
import com.seuprojeto.dto.LoginResponse;
import com.seuprojeto.dto.RegisterRequest;
import com.seuprojeto.model.User;
import com.seuprojeto.repository.UserRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
public class AuthService {
    
    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    private final JwtService jwtService;
    private final AuthenticationManager authenticationManager;
    
    public LoginResponse login(LoginRequest request) {
        authenticationManager.authenticate(
            new UsernamePasswordAuthenticationToken(
                request.getUsername(),
                request.getPassword()
            )
        );
        
        User user = userRepository.findByUsername(request.getUsername())
            .orElseThrow(() -> new RuntimeException("Usuário não encontrado"));
        
        String token = jwtService.generateToken(user);
        
        return new LoginResponse(token, user.getUsername(), user.getRole().name());
    }
    
    public User register(RegisterRequest request) {
        if (userRepository.existsByUsername(request.getUsername())) {
            throw new RuntimeException("Username já existe");
        }
        
        User user = new User();
        user.setUsername(request.getUsername());
        user.setPassword(passwordEncoder.encode(request.getPassword()));
        user.setEmail(request.getEmail());
        user.setRole(request.getRole());
        
        return userRepository.save(user);
    }
}
```

### PASSO 11: JwtAuthenticationFilter.java

```java
package com.seuprojeto.security;

import com.seuprojeto.service.JwtService;
import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.RequiredArgsConstructor;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.web.authentication.WebAuthenticationDetailsSource;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;

@Component
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    
    private final JwtService jwtService;
    private final UserDetailsService userDetailsService;
    
    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain
    ) throws ServletException, IOException {
        
        final String authHeader = request.getHeader("Authorization");
        final String jwt;
        final String username;
        
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);
            return;
        }
        
        jwt = authHeader.substring(7);
        username = jwtService.extractUsername(jwt);
        
        if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
            UserDetails userDetails = this.userDetailsService.loadUserByUsername(username);
            
            if (jwtService.validateToken(jwt, userDetails)) {
                UsernamePasswordAuthenticationToken authToken = new UsernamePasswordAuthenticationToken(
                    userDetails,
                    null,
                    userDetails.getAuthorities()
                );
                authToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                SecurityContextHolder.getContext().setAuthentication(authToken);
            }
        }
        filterChain.doFilter(request, response);
    }
}
```

### PASSO 12: SecurityConfig.java

```java
package com.seuprojeto.config;

import com.seuprojeto.repository.UserRepository;
import com.seuprojeto.security.JwtAuthenticationFilter;
import lombok.RequiredArgsConstructor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.AuthenticationProvider;
import org.springframework.security.authentication.dao.DaoAuthenticationProvider;
import org.springframework.security.config.annotation.authentication.configuration.AuthenticationConfiguration;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;
import org.springframework.web.cors.CorsConfiguration;
import org.springframework.web.cors.CorsConfigurationSource;
import org.springframework.web.cors.UrlBasedCorsConfigurationSource;

import java.util.List;

@Configuration
@EnableWebSecurity
@EnableMethodSecurity
@RequiredArgsConstructor
public class SecurityConfig {
    
    private final JwtAuthenticationFilter jwtAuthFilter;
    private final UserRepository userRepository;
    
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/user/**").hasAnyRole("USER", "ADMIN")
                .anyRequest().authenticated()
            )
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            .authenticationProvider(authenticationProvider())
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);
        
        return http.build();
    }
    
    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration configuration = new CorsConfiguration();
        configuration.setAllowedOrigins(List.of("http://localhost:4200", "http://seu-bucket-s3.s3-website-us-east-1.amazonaws.com"));
        configuration.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
        configuration.setAllowedHeaders(List.of("*"));
        configuration.setAllowCredentials(true);
        
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", configuration);
        return source;
    }
    
    @Bean
    public UserDetailsService userDetailsService() {
        return username -> userRepository.findByUsername(username)
            .orElseThrow(() -> new UsernameNotFoundException("Usuário não encontrado"));
    }
    
    @Bean
    public AuthenticationProvider authenticationProvider() {
        DaoAuthenticationProvider authProvider = new DaoAuthenticationProvider();
        authProvider.setUserDetailsService(userDetailsService());
        authProvider.setPasswordEncoder(passwordEncoder());
        return authProvider;
    }
    
    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

### PASSO 13: AuthController.java

```java
package com.seuprojeto.controller;

import com.seuprojeto.dto.LoginRequest;
import com.seuprojeto.dto.LoginResponse;
import com.seuprojeto.dto.RegisterRequest;
import com.seuprojeto.model.User;
import com.seuprojeto.service.AuthService;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/auth")
@RequiredArgsConstructor
public class AuthController {
    
    private final AuthService authService;
    
    @PostMapping("/login")
    public ResponseEntity<LoginResponse> login(@RequestBody LoginRequest request) {
        return ResponseEntity.ok(authService.login(request));
    }
    
    @PostMapping("/register")
    public ResponseEntity<User> register(@RequestBody RegisterRequest request) {
        return ResponseEntity.ok(authService.register(request));
    }
}
```

### PASSO 14: UserController.java (Exemplo de rotas protegidas)

```java
package com.seuprojeto.controller;

import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/api")
public class UserController {
    
    @GetMapping("/user/dashboard")
    public ResponseEntity<String> userDashboard() {
        return ResponseEntity.ok("Dashboard do Usuário");
    }
    
    @GetMapping("/admin/dashboard")
    public ResponseEntity<String> adminDashboard() {
        return ResponseEntity.ok("Dashboard do Admin");
    }
}
```

---

## 🎨 FRONT-END (ANGULAR)

### PASSO 1: Criar Projeto Angular

```bash
ng new frontend
cd frontend
```

### PASSO 2: Estrutura de Pastas

```
src/app/
├── auth/
│   ├── login/
│   ├── register/
│   ├── auth.service.ts
│   └── auth.guard.ts
├── interceptors/
│   └── jwt.interceptor.ts
├── models/
│   └── user.model.ts
├── pages/
│   ├── dashboard/
│   └── admin/
└── app.routes.ts
```

### PASSO 3: Models (user.model.ts)

```typescript
export interface LoginRequest {
  username: string;
  password: string;
}

export interface LoginResponse {
  token: string;
  username: string;
  role: string;
}

export interface User {
  id: number;
  username: string;
  email: string;
  role: string;
}
```

### PASSO 4: AuthService (auth.service.ts)

```typescript
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable, BehaviorSubject } from 'rxjs';
import { tap } from 'rxjs/operators';
import { LoginRequest, LoginResponse } from '../models/user.model';
import { Router } from '@angular/router';

@Injectable({
  providedIn: 'root'
})
export class AuthService {
  private apiUrl = 'http://localhost:8080/api/auth';
  private tokenKey = 'auth_token';
  private userKey = 'user_data';
  
  private isAuthenticatedSubject = new BehaviorSubject<boolean>(this.hasToken());
  public isAuthenticated$ = this.isAuthenticatedSubject.asObservable();

  constructor(private http: HttpClient, private router: Router) {}

  login(credentials: LoginRequest): Observable<LoginResponse> {
    return this.http.post<LoginResponse>(`${this.apiUrl}/login`, credentials)
      .pipe(
        tap(response => {
          localStorage.setItem(this.tokenKey, response.token);
          localStorage.setItem(this.userKey, JSON.stringify(response));
          this.isAuthenticatedSubject.next(true);
        })
      );
  }

  logout(): void {
    localStorage.removeItem(this.tokenKey);
    localStorage.removeItem(this.userKey);
    this.isAuthenticatedSubject.next(false);
    this.router.navigate(['/login']);
  }

  getToken(): string | null {
    return localStorage.getItem(this.tokenKey);
  }

  getUserData(): LoginResponse | null {
    const data = localStorage.getItem(this.userKey);
    return data ? JSON.parse(data) : null;
  }

  hasToken(): boolean {
    return !!this.getToken();
  }

  isAdmin(): boolean {
    const user = this.getUserData();
    return user?.role === 'ADMIN';
  }
}
```

### PASSO 5: JWT Interceptor (jwt.interceptor.ts)

```typescript
import { HttpInterceptorFn } from '@angular/common/http';
import { inject } from '@angular/core';
import { AuthService } from '../auth/auth.service';

export const jwtInterceptor: HttpInterceptorFn = (req, next) => {
  const authService = inject(AuthService);
  const token = authService.getToken();
  
  if (token && !req.url.includes('/auth/')) {
    req = req.clone({
      setHeaders: {
        Authorization: `Bearer ${token}`
      }
    });
  }
  
  return next(req);
};
```

### PASSO 6: Auth Guard (auth.guard.ts)

```typescript
import { inject } from '@angular/core';
import { Router } from '@angular/router';
import { AuthService } from './auth.service';

export const authGuard = () => {
  const authService = inject(AuthService);
  const router = inject(Router);

  if (authService.hasToken()) {
    return true;
  }

  router.navigate(['/login']);
  return false;
};

export const adminGuard = () => {
  const authService = inject(AuthService);
  const router = inject(Router);

  if (authService.hasToken() && authService.isAdmin()) {
    return true;
  }

  router.navigate(['/dashboard']);
  return false;
};
```

### PASSO 7: App Routes (app.routes.ts)

```typescript
import { Routes } from '@angular/router';
import { authGuard, adminGuard } from './auth/auth.guard';

export const routes: Routes = [
  { path: '', redirectTo: '/login', pathMatch: 'full' },
  { 
    path: 'login', 
    loadComponent: () => import('./auth/login/login.component').then(m => m.LoginComponent)
  },
  { 
    path: 'dashboard', 
    loadComponent: () => import('./pages/dashboard/dashboard.component').then(m => m.DashboardComponent),
    canActivate: [authGuard]
  },
  { 
    path: 'admin', 
    loadComponent: () => import('./pages/admin/admin.component').then(m => m.AdminComponent),
    canActivate: [authGuard, adminGuard]
  },
  { path: '**', redirectTo: '/login' }
];
```

### PASSO 8: App Config (app.config.ts)

```typescript
import { ApplicationConfig, provideZoneChangeDetection } from '@angular/core';
import { provideRouter } from '@angular/router';
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { routes } from './app.routes';
import { jwtInterceptor } from './interceptors/jwt.interceptor';

export const appConfig: ApplicationConfig = {
  providers: [
    provideZoneChangeDetection({ eventCoalescing: true }),
    provideRouter(routes),
    provideHttpClient(withInterceptors([jwtInterceptor]))
  ]
};
```

### PASSO 9: Login Component (login.component.ts)

```typescript
import { Component } from '@angular/core';
import { CommonModule } from '@angular/common';
import { FormsModule } from '@angular/forms';
import { Router } from '@angular/router';
import { AuthService } from '../auth.service';
import { LoginRequest } from '../../models/user.model';

@Component({
  selector: 'app-login',
  standalone: true,
  imports: [CommonModule, FormsModule],
  template: `
    <div class="login-container">
      <form (ngSubmit)="onLogin()" class="login-form">
        <h2>Login</h2>
        
        <div class="form-group">
          <label>Usuário</label>
          <input type="text" [(ngModel)]="credentials.username" name="username" required>
        </div>
        
        <div class="form-group">
          <label>Senha</label>
          <input type="password" [(ngModel)]="credentials.password" name="password" required>
        </div>
        
        <button type="submit">Entrar</button>
        
        <div *ngIf="errorMessage" class="error">{{ errorMessage }}</div>
      </form>
    </div>
  `,
  styles: [`
    .login-container {
      display: flex;
      justify-content: center;
      align-items: center;
      height: 100vh;
      background: #f5f5f5;
    }
    
    .login-form {
      background: white;
      padding: 2rem;
      border-radius: 8px;
      box-shadow: 0 2px 10px rgba(0,0,0,0.1);
      width: 100%;
      max-width: 400px;
    }
    
    .form-group {
      margin-bottom: 1rem;
    }
    
    label {
      display: block;
      margin-bottom: 0.5rem;
      font-weight: bold;
    }
    
    input {
      width: 100%;
      padding: 0.5rem;
      border: 1px solid #ddd;
      border-radius: 4px;
    }
    
    button {
      width: 100%;
      padding: 0.75rem;
      background: #007bff;
      color: white;
      border: none;
      border-radius: 4px;
      cursor: pointer;
      font-size: 1rem;
    }
    
    button:hover {
      background: #0056b3;
    }
    
    .error {
      color: red;
      margin-top: 1rem;
    }
  `]
})
export class LoginComponent {
  credentials: LoginRequest = {
    username: '',
    password: ''
  };
  
  errorMessage = '';

  constructor(
    private authService: AuthService,
    private router: Router
  ) {}

  onLogin(): void {
    this.authService.login(this.credentials).subscribe({
      next: (response) => {
        if (response.role === 'ADMIN') {
          this.router.navigate(['/admin']);
        } else {
          this.router.navigate(['/dashboard']);
        }
      },
      error: (error) => {
        this.errorMessage = 'Credenciais inválidas';
        console.error('Erro no login:', error);
      }
    });
  }
}
```

### PASSO 10: Dashboard Component (dashboard.component.ts)

```typescript
import { Component, OnInit } from '@angular/core';
import { CommonModule } from '@angular/common';
import { AuthService } from '../../auth/auth.service';
import { LoginResponse } from '../../models/user.model';

@Component({
  selector: 'app-dashboard',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div class="dashboard">
      <nav>
        <h1>Dashboard do Usuário</h1>
        <button (click)="logout()">Sair</button>
      </nav>
      
      <div class="content">
        <h2>Bem-vindo, {{ userData?.username }}!</h2>
        <p>Role: {{ userData?.role }}</p>
      </div>
    </div>
  `,
  styles: [`
    .dashboard {
      min-height: 100vh;
    }
    
    nav {
      display: flex;
      justify-content: space-between;
      align-items: center;
      padding: 1rem 2rem;
      background: #007bff;
      color: white;
    }
    
    .content {
      padding: 2rem;
    }
    
    button {
      padding: 0.5rem 1rem;
      background: white;
      color: #007bff;
      border: none;
      border-radius: 4px;
      cursor: pointer;
    }
  `]
})
export class DashboardComponent implements OnInit {
  userData: LoginResponse | null = null;

  constructor(private authService: AuthService) {}

  ngOnInit(): void {
    this.userData = this.authService.getUserData();
  }

  logout(): void {
    this.authService.logout();
  }
}
```

### PASSO 11: Admin Component (admin.component.ts)

Similar ao Dashboard, mas com conteúdo específico de admin.

---

## ☁️ DEPLOY NA AWS

### OPÇÃO 1: AWS Academy (Mais Comum)

#### Back-End (EC2)

1. **Criar Instância EC2**
   - Login no AWS Academy Learner Lab
   - Services → EC2 → Launch Instance
   - Nome: `backend-spring`
   - AMI: Amazon Linux 2023
   - Instance type: t2.micro
   - Key pair: Criar novo ou usar existente
   - Security Group: Permitir portas 22 (SSH) e 8080 (Spring Boot)

2. **Conectar via SSH**
```bash
chmod 400 sua-chave.pem
ssh -i sua-chave.pem ec2-user@seu-ip-publico
```

3. **Instalar Java 17**
```bash
sudo yum update -y
sudo yum install java-17-amazon-corretto-headless -y
java -version
```

4. **Subir a Aplicação**
```bash
# No seu computador, gere o JAR
mvn clean package -DskipTests

# Envie para EC2
scp -i sua-chave.pem target/seu-app.jar ec2-user@seu-ip:/home/ec2-user/

# No EC2, configure variáveis de ambiente
export JWT_SECRET="sua-chave-secreta-super-segura-256-bits"
export SPRING_DATASOURCE_URL="jdbc:postgresql://seu-rds:5432/seudb"
export SPRING_DATASOURCE_USERNAME="postgres"
export SPRING_DATASOURCE_PASSWORD="senha"

# Execute
java -jar seu-app.jar
```

5. **Rodar em Background (pm2 ou nohup)**
```bash
# Com nohup
nohup java -jar seu-app.jar > app.log 2>&1 &

# Ou instalar pm2
sudo npm install -g pm2
pm2 start "java -jar seu-app.jar" --name backend
pm2 save
pm2 startup
```

#### Front-End (S3 + CloudFront)

1. **Build do Angular**
```bash
ng build --configuration production
```

2. **Criar Bucket S3**
   - S3 → Create bucket
   - Nome único: `seu-projeto-frontend`
   - Desmarcar "Block all public access"
   - Create bucket

3. **Configurar Bucket para Hosting**
   - Properties → Static website hosting → Enable
   - Index document: `index.html`
   - Error document: `index.html`

4. **Política do Bucket**
   - Permissions → Bucket Policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::seu-projeto-frontend/*"
    }
  ]
}
```

5. **Upload dos Arquivos**
```bash
aws s3 sync dist/seu-projeto/browser s3://seu-projeto-frontend
```

6. **Atualizar CORS no Back-End**
   - Adicione a URL do S3 no `SecurityConfig.java`:
```java
configuration.setAllowedOrigins(List.of(
    "http://localhost:4200",
    "http://seu-projeto-frontend.s3-website-us-east-1.amazonaws.com"
));
```

#### Banco de Dados (RDS - Opcional)

Se quiser usar RDS ao invés de banco local:

1. RDS → Create database
2. PostgreSQL
3. Free tier
4. Configure credentials
5. Security group: Permitir porta 5432 do EC2

---

## ✅ TESTES FINAIS

### Checklist de Validação

#### Back-End
- [ ] Senhas criptografadas no banco (verificar com `SELECT password FROM users`)
- [ ] JWT secret no `.env` ou `application.properties`
- [ ] Endpoint `/api/auth/login` retorna token JWT
- [ ] Rotas `/api/user/*` bloqueadas sem token
- [ ] Rotas `/api/admin/*` bloqueadas para usuários sem role ADMIN

#### Front-End
- [ ] Tentar acessar `/dashboard` sem login (deve redirecionar para `/login`)
- [ ] Token anexado automaticamente nas requisições (verificar no DevTools → Network)
- [ ] Logout remove token e redireciona

#### AWS
- [ ] Front-end acessível via URL do S3
- [ ] Back-end respondendo no EC2 (testar com Postman no IP:8080)
- [ ] Integração funcionando (login funciona pelo front hospedado)

### Script de Teste (Postman)

```bash
# 1. Registrar usuário
POST http://seu-ec2:8080/api/auth/register
{
  "username": "admin",
  "password": "senha123",
  "email": "admin@teste.com",
  "role": "ADMIN"
}

# 2. Login
POST http://seu-ec2:8080/api/auth/login
{
  "username": "admin",
  "password": "senha123"
}

# 3. Testar rota protegida
GET http://seu-ec2:8080/api/admin/dashboard
Header: Authorization: Bearer SEU_TOKEN_JWT
```

---

## 📝 OBSERVAÇÕES IMPORTANTES

### Pontuação do Projeto
- Senhas criptografadas: 10%
- JWT no .env: 5%
- Endpoint login: 15%
- Spring Security bloqueando rotas: 15%
- Autorização por roles: 5%
- CanActivate no Angular: 10%
- HttpInterceptor: 10%
- Deploy na AWS: 10%
- **TOTAL: 100%**

### Dicas para Apresentação
1. Mostre o código do `SecurityConfig.java`
2. Mostre o arquivo `.env`
3. Demonstre login e acesso a rotas
4. Mostre tentativa de acesso sem token
5. Mostre aplicação rodando na nuvem

### Links Úteis
- [Spring Security Docs](https://spring.io/guides/gs/securing-web/)
- [JWT.io](https://jwt.io/)
- [Angular Guards](https://angular.io/guide/router#preventing-unauthorized-access)
- [AWS Academy Guide](https://awsacademy.instructure.com/)

---

## 🆘 TROUBLESHOOTING

### Erro: JWT secret muito curta
```
Error: The specified key byte array is 168 bits which is not secure enough
```
**Solução**: Use um secret com pelo menos 256 bits (32 caracteres)

### Erro: CORS no Angular
```
Access to XMLHttpRequest has been blocked by CORS policy
```
**Solução**: Verifique `corsConfigurationSource()` no `SecurityConfig.java`

### EC2 não aceita conexões
**Solução**: Verifique Security Group → Inbound Rules → Porta 8080

### S3 não carrega página
**Solução**: 
1. Bucket Policy configurada?
2. Static website hosting habilitado?
3. `index.html` existe no bucket?

---

**BOA SORTE! 🚀**
