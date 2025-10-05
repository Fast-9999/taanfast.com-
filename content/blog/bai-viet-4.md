---
title: "Xây dựng REST API Java 'Thực chiến' với Spring Boot & JWT Auth: Từ 0 đến Token"
date: 2025-10-05
featured_image: "https://www.mytaskpanel.com/wp-content/uploads/2023/04/consulting-blog-09.webp"
description: "Thực chiến Spring Boot từ con số 0: xây dựng REST API CRUD, bảo mật bằng JWT Authentication, test với Postman và deploy lên Heroku — hướng dẫn từng bước cho backend developer muốn có project thực tế trong portfolio."
categories: ["Java", "Backend"]
tags:
  [
    "java",
    "spring-boot",
    "rest-api",
    "jwt",
    "authentication",
    "crud",
    "backend-development",
    "heroku-deploy",
    "portfolio-project",
  ]
slug: "spring-boot-rest-jwt-tutorial"
draft: false
---

# Xây dựng REST API Java 'Thực chiến' với Spring Boot & JWT Auth: Từ 0 đến Token


## Giới thiệu: Nâng cấp Portfolio Backend của bạn

Nếu bạn là một Backend Developer tập sự hoặc sinh viên muốn có một dự án thực tế, đây là bài hướng dẫn dành cho bạn. Chúng ta sẽ xây dựng REST API CRUD đầy đủ (Create, Read, Update, Delete) và bảo mật bằng JSON Web Token (JWT) — một chuẩn authentication hiện đại. Sau cùng, bạn sẽ biết cách đóng gói và triển khai ứng dụng lên cloud (Heroku hoặc Somee), tạo thành một API thật sự “production-ready”.

---

## 🎯 Mục tiêu dự án

Xây dựng API quản lý bài viết/sản phẩm (CRUD) với:

* **Kiến trúc chuẩn:** Controller / Service / Repository.
* **Validation & DTO:** kiểm soát input và tránh lộ dữ liệu nhạy cảm.
* **Bảo mật:** Đăng ký, đăng nhập, xác thực JWT.
* **Triển khai:** Deploy app `.jar` lên Heroku hoặc Somee.

Repo mẫu: `spring-boot-starter-api` + Postman collection.

---

## 1. Khởi tạo Project — Spring Initializr

Truy cập [start.spring.io](https://start.spring.io/) và cấu hình:

| Trường           | Giá trị                                | Giải thích                 |
| ---------------- | -------------------------------------- | -------------------------- |
| Project          | Maven Project                          | Công cụ build              |
| Language         | Java                                   | Ngôn ngữ chính             |
| Spring Boot      | 3.2.x                                  | Phiên bản ổn định mới nhất |
| Group / Artifact | `com.example` / `spring-boot-rest-jwt` | Định danh project          |
| Java Version     | 17 hoặc 21                             | LTS version                |

### Dependencies cần thiết:

* Spring Web — REST API.
* Spring Data JPA — ORM.
* H2 Database — phát triển cục bộ.
* Spring Security — bảo mật.
* Validation — @Valid, @NotNull, ...
* Lombok — giảm boilerplate.
* `jjwt` (API, impl, jackson) — JWT auth.

Tải project và mở trong IDE.

---

## 2. Cấu trúc Project & API CRUD

```
com.example.springbootrestjwt
├── config
├── controller
├── dto
├── entity
├── repository
├── service
└── exception
```

### 2.1 Entity: `User` (cho JWT Authentication)

```java
@Entity
@Data
@NoArgsConstructor
@AllArgsConstructor
@Table(name = "users")
public class User {
  @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  @Column(nullable = false, unique = true)
  private String username;

  @Column(nullable = false)
  private String password;

  @Column(nullable = false, unique = true)
  private String email;

  private String firstName;
  private String lastName;
  private Instant createdAt = Instant.now();
  private Instant updatedAt;

  @Enumerated(EnumType.STRING)
  private Role role = Role.USER;
}

public enum Role {
  USER, ADMIN
}
```

### 2.2 Entity: `Post`

```java
@Entity
@Data
@NoArgsConstructor
@AllArgsConstructor
@Table(name = "posts")
public class Post {
  @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  @Column(nullable = false, unique = true)
  private String title;

  @Column(columnDefinition = "TEXT")
  private String content;

  private String author;
  private Instant createdAt = Instant.now();
  private Instant updatedAt;

  @ManyToOne(fetch = FetchType.LAZY)
  @JoinColumn(name = "user_id")
  private User user;
}
```

### 2.3 Repository Layer

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
  Optional<User> findByUsername(String username);
  Optional<User> findByEmail(String email);
  boolean existsByUsername(String username);
  boolean existsByEmail(String email);
}

@Repository
public interface PostRepository extends JpaRepository<Post, Long> {
  Optional<Post> findByTitle(String title);
  List<Post> findByUser(User user);
  List<Post> findByAuthor(String author);
}
```

### 2.4 Service Layer

```java
@Service
@RequiredArgsConstructor
public class UserService {
  private final UserRepository userRepository;
  private final PasswordEncoder passwordEncoder;

  public User registerUser(RegisterRequest request) {
    if (userRepository.existsByUsername(request.getUsername())) {
      throw new UserAlreadyExistsException("Username already exists");
    }
    if (userRepository.existsByEmail(request.getEmail())) {
      throw new UserAlreadyExistsException("Email already exists");
    }

    User user = new User();
    user.setUsername(request.getUsername());
    user.setEmail(request.getEmail());
    user.setPassword(passwordEncoder.encode(request.getPassword()));
    user.setFirstName(request.getFirstName());
    user.setLastName(request.getLastName());
    
    return userRepository.save(user);
  }

  public User findByUsername(String username) {
    return userRepository.findByUsername(username)
      .orElseThrow(() -> new UserNotFoundException("User not found"));
  }
}

@Service
@RequiredArgsConstructor
public class PostService {
  private final PostRepository postRepository;

  public List<Post> getAllPosts() { return postRepository.findAll(); }

  public Post getPostById(Long id) {
    return postRepository.findById(id)
      .orElseThrow(() -> new ResourceNotFoundException("Post not found with id: " + id));
  }

  public Post createPost(PostRequestDTO dto, User user) {
    Post post = new Post();
    post.setTitle(dto.getTitle());
    post.setContent(dto.getContent());
    post.setAuthor(dto.getAuthor());
    post.setUser(user);
    return postRepository.save(post);
  }

  public Post updatePost(Long id, PostRequestDTO dto, User user) {
    Post post = getPostById(id);
    if (!post.getUser().getId().equals(user.getId())) {
      throw new UnauthorizedException("You can only update your own posts");
    }
    
    post.setTitle(dto.getTitle());
    post.setContent(dto.getContent());
    post.setAuthor(dto.getAuthor());
    post.setUpdatedAt(Instant.now());
    return postRepository.save(post);
  }

  public void deletePost(Long id, User user) {
    Post post = getPostById(id);
    if (!post.getUser().getId().equals(user.getId())) {
      throw new UnauthorizedException("You can only delete your own posts");
    }
    postRepository.deleteById(id);
  }
}
```

### 2.5 Controller Layer

```java
@RestController
@RequestMapping("/api/v1/auth")
@RequiredArgsConstructor
public class AuthController {
  private final UserService userService;
  private final JwtService jwtService;
  private final AuthenticationManager authenticationManager;

  @PostMapping("/register")
  public ResponseEntity<AuthResponse> register(@Valid @RequestBody RegisterRequest request) {
    User user = userService.registerUser(request);
    String token = jwtService.generateToken(user.getUsername());
    
    AuthResponse response = new AuthResponse(token, user.getUsername(), user.getEmail());
    return ResponseEntity.status(HttpStatus.CREATED).body(response);
  }

  @PostMapping("/login")
  public ResponseEntity<AuthResponse> login(@Valid @RequestBody LoginRequest request) {
    authenticationManager.authenticate(
      new UsernamePasswordAuthenticationToken(request.getUsername(), request.getPassword())
    );
    
    User user = userService.findByUsername(request.getUsername());
    String token = jwtService.generateToken(user.getUsername());
    
    AuthResponse response = new AuthResponse(token, user.getUsername(), user.getEmail());
    return ResponseEntity.ok(response);
  }
}

@RestController
@RequestMapping("/api/v1/posts")
@RequiredArgsConstructor
public class PostController {
  private final PostService postService;

  @GetMapping
  public ResponseEntity<List<Post>> getAll() {
    return ResponseEntity.ok(postService.getAllPosts());
  }

  @GetMapping("/{id}")
  public ResponseEntity<Post> getById(@PathVariable Long id) {
    return ResponseEntity.ok(postService.getPostById(id));
  }

  @PostMapping
  @PreAuthorize("hasRole('USER')")
  public ResponseEntity<Post> create(@Valid @RequestBody PostRequestDTO dto, 
                                   Authentication authentication) {
    User user = (User) authentication.getPrincipal();
    Post created = postService.createPost(dto, user);
    return ResponseEntity.status(HttpStatus.CREATED).body(created);
  }

  @PutMapping("/{id}")
  @PreAuthorize("hasRole('USER')")
  public ResponseEntity<Post> update(@PathVariable Long id, 
                                   @Valid @RequestBody PostRequestDTO dto,
                                   Authentication authentication) {
    User user = (User) authentication.getPrincipal();
    Post updated = postService.updatePost(id, dto, user);
    return ResponseEntity.ok(updated);
  }

  @DeleteMapping("/{id}")
  @PreAuthorize("hasRole('USER')")
  public ResponseEntity<Void> delete(@PathVariable Long id, Authentication authentication) {
    User user = (User) authentication.getPrincipal();
    postService.deletePost(id, user);
    return ResponseEntity.noContent().build();
  }
}
```

---

## 3. DTO & Validation

### 3.1 Auth DTOs

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class RegisterRequest {
  @NotBlank @Size(min = 3, max = 20)
  private String username;
  
  @NotBlank @Email
  private String email;
  
  @NotBlank @Size(min = 6, max = 100)
  private String password;
  
  @Size(max = 50)
  private String firstName;
  
  @Size(max = 50)
  private String lastName;
}

@Data
@NoArgsConstructor
@AllArgsConstructor
public class LoginRequest {
  @NotBlank
  private String username;
  
  @NotBlank
  private String password;
}

@Data
@NoArgsConstructor
@AllArgsConstructor
public class AuthResponse {
  private String token;
  private String username;
  private String email;
  private String type = "Bearer";
}
```

### 3.2 Post DTOs

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class PostRequestDTO {
  @NotBlank @Size(min = 5, max = 100)
  private String title;
  @NotBlank
  private String content;
  @Size(max = 50)
  private String author;
}

@Data
@NoArgsConstructor
@AllArgsConstructor
public class PostResponseDTO {
  private Long id;
  private String title;
  private String content;
  private String author;
  private String username;
  private Instant createdAt;
  private Instant updatedAt;
}
```

---

## 4. Bảo mật với Spring Security & JWT

### 4.1 Security Configuration

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity(prePostEnabled = true)
@RequiredArgsConstructor
public class SecurityConfig {
  private final JwtAuthFilter jwtAuthFilter;
  private final UserDetailsService userDetailsService;

  @Bean
  public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
      .csrf(AbstractHttpConfigurer::disable)
      .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
      .authorizeHttpRequests(auth -> auth
        .requestMatchers("/api/v1/auth/**").permitAll()
        .requestMatchers("/h2-console/**").permitAll()
        .anyRequest().authenticated()
      )
      .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class)
      .headers(headers -> headers.frameOptions().disable()); // For H2 Console
    
    return http.build();
  }

  @Bean
  public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
  }

  @Bean
  public AuthenticationManager authenticationManager(AuthenticationConfiguration config) throws Exception {
    return config.getAuthenticationManager();
  }
}
```

### 4.2 JWT Service (Complete Implementation)

```java
@Service
@RequiredArgsConstructor
public class JwtService {
  @Value("${jwt.secret}")
  private String secretKey;
  
  @Value("${jwt.expiration}")
  private long jwtExpiration;

  public String generateToken(String username) {
    return Jwts.builder()
      .setSubject(username)
      .setIssuedAt(new Date())
      .setExpiration(new Date(System.currentTimeMillis() + jwtExpiration))
      .signWith(getSignInKey(), SignatureAlgorithm.HS256)
      .compact();
  }

  public boolean validateToken(String token) {
    try {
      Jwts.parserBuilder()
        .setSigningKey(getSignInKey())
        .build()
        .parseClaimsJws(token);
      return true;
    } catch (JwtException | IllegalArgumentException e) {
      return false;
    }
  }

  public String extractUsername(String token) {
    return extractClaim(token, Claims::getSubject);
  }

  public <T> T extractClaim(String token, Function<Claims, T> claimsResolver) {
    final Claims claims = extractAllClaims(token);
    return claimsResolver.apply(claims);
  }

  private Claims extractAllClaims(String token) {
    return Jwts.parserBuilder()
      .setSigningKey(getSignInKey())
      .build()
      .parseClaimsJws(token)
      .getBody();
  }

  private Key getSignInKey() {
    byte[] keyBytes = Decoders.BASE64.decode(secretKey);
    return Keys.hmacShaKeyFor(keyBytes);
  }

  public Authentication getAuthentication(String token) {
    String username = extractUsername(token);
    UserDetails userDetails = userDetailsService.loadUserByUsername(username);
    return new UsernamePasswordAuthenticationToken(userDetails, null, userDetails.getAuthorities());
  }
}
```

### 4.3 JWT Filter (Complete Implementation)

```java
@Component
@RequiredArgsConstructor
public class JwtAuthFilter extends OncePerRequestFilter {
  private final JwtService jwtService;
  private final UserDetailsService userDetailsService;

  @Override
  protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
      throws ServletException, IOException {
    
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
      UserDetails userDetails = userDetailsService.loadUserByUsername(username);
      
      if (jwtService.validateToken(jwt) && jwtService.validateToken(jwt)) {
        UsernamePasswordAuthenticationToken authToken = new UsernamePasswordAuthenticationToken(
          userDetails, null, userDetails.getAuthorities()
        );
        authToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
        SecurityContextHolder.getContext().setAuthentication(authToken);
      }
    }
    
    filterChain.doFilter(request, response);
  }
}
```

---

## 5. Configuration & Exception Handling

### 5.1 Application Properties

```properties
# Database Configuration
spring.datasource.url=jdbc:h2:mem:testdb
spring.datasource.driverClassName=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=password

# H2 Console (for development)
spring.h2.console.enabled=true
spring.h2.console.path=/h2-console

# JPA Configuration
spring.jpa.database-platform=org.hibernate.dialect.H2Dialect
spring.jpa.hibernate.ddl-auto=create-drop
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true

# JWT Configuration
jwt.secret=404E635266556A586E3272357538782F413F4428472B4B6250645367566B5970
jwt.expiration=86400000

# Server Configuration
server.port=8080
server.servlet.context-path=/

# Logging
logging.level.com.example.springbootrestjwt=DEBUG
logging.level.org.springframework.security=DEBUG
```

### 5.2 Global Exception Handler

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

  @ExceptionHandler(ResourceNotFoundException.class)
  public ResponseEntity<ErrorResponse> handleResourceNotFound(ResourceNotFoundException ex) {
    ErrorResponse error = new ErrorResponse("RESOURCE_NOT_FOUND", ex.getMessage());
    return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
  }

  @ExceptionHandler(UserAlreadyExistsException.class)
  public ResponseEntity<ErrorResponse> handleUserAlreadyExists(UserAlreadyExistsException ex) {
    ErrorResponse error = new ErrorResponse("USER_ALREADY_EXISTS", ex.getMessage());
    return ResponseEntity.status(HttpStatus.CONFLICT).body(error);
  }

  @ExceptionHandler(UnauthorizedException.class)
  public ResponseEntity<ErrorResponse> handleUnauthorized(UnauthorizedException ex) {
    ErrorResponse error = new ErrorResponse("UNAUTHORIZED", ex.getMessage());
    return ResponseEntity.status(HttpStatus.UNAUTHORIZED).body(error);
  }

  @ExceptionHandler(MethodArgumentNotValidException.class)
  public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
    String message = ex.getBindingResult().getFieldErrors().stream()
      .map(error -> error.getField() + ": " + error.getDefaultMessage())
      .collect(Collectors.joining(", "));
    
    ErrorResponse error = new ErrorResponse("VALIDATION_ERROR", message);
    return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(error);
  }

  @ExceptionHandler(Exception.class)
  public ResponseEntity<ErrorResponse> handleGeneric(Exception ex) {
    log.error("Unexpected error occurred", ex);
    ErrorResponse error = new ErrorResponse("INTERNAL_SERVER_ERROR", "An unexpected error occurred");
    return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(error);
  }
}

@Data
@AllArgsConstructor
@NoArgsConstructor
class ErrorResponse {
  private String error;
  private String message;
  private Instant timestamp = Instant.now();
}
```

### 5.3 Custom Exceptions

```java
public class ResourceNotFoundException extends RuntimeException {
  public ResourceNotFoundException(String message) {
    super(message);
  }
}

public class UserAlreadyExistsException extends RuntimeException {
  public UserAlreadyExistsException(String message) {
    super(message);
  }
}

public class UserNotFoundException extends RuntimeException {
  public UserNotFoundException(String message) {
    super(message);
  }
}

public class UnauthorizedException extends RuntimeException {
  public UnauthorizedException(String message) {
    super(message);
  }
}
```

---

## 6. Testing API (Postman)

### 6.1 Authentication Flow

**Step 1: Register User**
```http
POST http://localhost:8080/api/v1/auth/register
Content-Type: application/json

{
  "username": "taansfast",
  "email": "taansfast@example.com",
  "password": "password123",
  "firstName": "Taans",
  "lastName": "Fast"
}
```

**Response:**
```json
{
  "token": "eyJhbGciOiJIUzI1NiJ9...",
  "username": "taansfast",
  "email": "taansfast@example.com",
  "type": "Bearer"
}
```

**Step 2: Login**
```http
POST http://localhost:8080/api/v1/auth/login
Content-Type: application/json

{
  "username": "taansfast",
  "password": "password123"
}
```

**Step 3: Use Token in Headers**
```http
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
```

### 6.2 Post CRUD Operations

**Create Post:**
```http
POST http://localhost:8080/api/v1/posts
Authorization: Bearer {{jwt_token}}
Content-Type: application/json

{
  "title": "REST API Spring Boot Tutorial",
  "content": "Complete guide to building REST API with Spring Boot and JWT authentication.",
  "author": "TaansFast"
}
```

**Get All Posts:**
```http
GET http://localhost:8080/api/v1/posts
```

**Get Post by ID:**
```http
GET http://localhost:8080/api/v1/posts/1
```

**Update Post:**
```http
PUT http://localhost:8080/api/v1/posts/1
Authorization: Bearer {{jwt_token}}
Content-Type: application/json

{
  "title": "Updated: REST API Spring Boot Tutorial",
  "content": "Updated content with more details about JWT implementation.",
  "author": "TaansFast"
}
```

**Delete Post:**
```http
DELETE http://localhost:8080/api/v1/posts/1
Authorization: Bearer {{jwt_token}}
```

### 6.3 Error Handling Examples

**Validation Error:**
```json
{
  "error": "VALIDATION_ERROR",
  "message": "title: Title must be between 5 and 100 characters",
  "timestamp": "2024-01-15T10:30:00Z"
}
```

**Unauthorized Error:**
```json
{
  "error": "UNAUTHORIZED",
  "message": "You can only update your own posts",
  "timestamp": "2024-01-15T10:30:00Z"
}
```

---

## 7. Unit Testing

### 7.1 Test Dependencies

Thêm vào `pom.xml`:
```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-test</artifactId>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>org.springframework.security</groupId>
  <artifactId>spring-security-test</artifactId>
  <scope>test</scope>
</dependency>
```

### 7.2 Service Tests

```java
@ExtendWith(MockitoExtension.class)
class PostServiceTest {
  @Mock
  private PostRepository postRepository;
  
  @InjectMocks
  private PostService postService;

  @Test
  void shouldCreatePostSuccessfully() {
    // Given
    PostRequestDTO dto = new PostRequestDTO("Test Title", "Test Content", "Author");
    User user = new User();
    Post savedPost = new Post();
    savedPost.setId(1L);
    
    when(postRepository.save(any(Post.class))).thenReturn(savedPost);
    
    // When
    Post result = postService.createPost(dto, user);
    
    // Then
    assertThat(result.getId()).isEqualTo(1L);
    verify(postRepository).save(any(Post.class));
  }
}
```

### 7.3 Controller Tests

```java
@WebMvcTest(PostController.class)
class PostControllerTest {
  @Autowired
  private MockMvc mockMvc;
  
  @MockBean
  private PostService postService;

  @Test
  @WithMockUser
  void shouldCreatePost() throws Exception {
    PostRequestDTO dto = new PostRequestDTO("Test", "Content", "Author");
    Post post = new Post();
    post.setId(1L);
    
    when(postService.createPost(any(), any())).thenReturn(post);
    
    mockMvc.perform(post("/api/v1/posts")
        .contentType(MediaType.APPLICATION_JSON)
        .content(objectMapper.writeValueAsString(dto)))
        .andExpect(status().isCreated())
        .andExpect(jsonPath("$.id").value(1));
  }
}
```

---

## 8. Build & Deploy

### Build JAR

```bash
./mvnw clean package
java -jar target/app.jar
```

### Deploy Heroku

```bash
heroku create springboot-jwt-demo
heroku config:set JWT_SECRET=yourSecretKey
heroku deploy:jar target/app.jar
```

### Deploy Somee

Upload `.war` hoặc `.jar`, cấu hình DB trong dashboard.

---

---

## 9. Troubleshooting & Common Issues

### 9.1 Common Build Issues

**Issue: JWT Dependencies Not Found**
```bash
# Solution: Add to pom.xml
<dependency>
  <groupId>io.jsonwebtoken</groupId>
  <artifactId>jjwt-api</artifactId>
  <version>0.11.5</version>
</dependency>
<dependency>
  <groupId>io.jsonwebtoken</groupId>
  <artifactId>jjwt-impl</artifactId>
  <version>0.11.5</version>
</dependency>
<dependency>
  <groupId>io.jsonwebtoken</groupId>
  <artifactId>jjwt-jackson</artifactId>
  <version>0.11.5</version>
</dependency>
```

**Issue: H2 Console Not Accessible**
```properties
# Add to application.properties
spring.h2.console.enabled=true
spring.h2.console.path=/h2-console
spring.jpa.hibernate.ddl-auto=create-drop
```

**Issue: JWT Token Invalid**
```java
// Check JWT secret key length (must be 256-bit)
jwt.secret=404E635266556A586E3272357538782F413F4428472B4B6250645367566B5970
```

### 9.2 Runtime Issues

**Issue: Authentication Failed**
- Check username/password in login request
- Verify JWT token format: `Bearer <token>`
- Check token expiration time

**Issue: Authorization Denied**
- Ensure user has correct role
- Check `@PreAuthorize` annotations
- Verify JWT filter is working

**Issue: Database Connection**
- Check H2 database is running
- Verify connection URL in application.properties
- Check H2 console at `http://localhost:8080/h2-console`

### 9.3 Deployment Issues

**Issue: Heroku Build Failed**
```bash
# Check Java version compatibility
heroku config:set JAVA_VERSION=17
heroku config:set JWT_SECRET=yourSecretKey
```

**Issue: Port Binding**
```properties
# For production deployment
server.port=${PORT:8080}
```

---

## 10. Kết luận & Next Steps

### 10.1 Những gì bạn đã học được

✅ **Complete REST API** với Spring Boot  
✅ **JWT Authentication** và Authorization  
✅ **Layered Architecture** (Controller/Service/Repository)  
✅ **Data Validation** và Error Handling  
✅ **Unit Testing** với Mockito và MockMvc  
✅ **Production Deployment** trên Heroku  

### 10.2 Portfolio Project Checklist

- [ ] **Source Code** trên GitHub với README chi tiết
- [ ] **Live Demo** trên Heroku hoặc cloud platform
- [ ] **Postman Collection** để test API
- [ ] **Unit Tests** với coverage > 80%
- [ ] **API Documentation** với Swagger/OpenAPI
- [ ] **Docker Support** cho containerization

### 10.3 Advanced Features để nâng cấp

🚀 **Caching**: Redis cho session management  
🚀 **Rate Limiting**: Prevent API abuse  
🚀 **API Versioning**: v1, v2 endpoints  
🚀 **Database Migration**: Flyway hoặc Liquibase  
🚀 **Monitoring**: Actuator endpoints  
🚀 **Logging**: Structured logging với ELK stack  

### 10.4 Tài nguyên bổ sung

**Documentation:**
- [Spring Boot Official Docs](https://spring.io/projects/spring-boot)
- [Spring Security Reference](https://docs.spring.io/spring-security/reference/)
- [JWT.io](https://jwt.io/) - JWT Debugger

**Tools:**
- [Postman Collection](https://www.postman.com/) - API Testing
- [Heroku CLI](https://devcenter.heroku.com/articles/heroku-cli) - Deployment
- [H2 Console](http://localhost:8080/h2-console) - Database Management

**Thử thách tiếp theo:**
> Deploy thành công trong **30 phút** và chia sẻ link API live trên GitHub!

---

## 11. Final CTA

🎯 **Bạn đã hoàn thành REST API Java world-class!**

**Tài nguyên hoàn chỉnh:**
- ✅ Full Source Code: `spring-boot-rest-jwt`
- ✅ Postman Collection: `/tools/postman_collection.json`
- ✅ Unit Tests: `/src/test/java`
- ✅ Deployment Guide: Heroku + Somee

**Chia sẻ thành quả:**
- 📱 Tweet: "Just built a complete REST API with Spring Boot + JWT! 🚀 #Java #SpringBoot #JWT"
- 💼 LinkedIn: "Portfolio project: Production-ready REST API with authentication"
- 🐙 GitHub: Star và fork repository để ủng hộ

> **"Một developer giỏi không chỉ code chạy được, mà còn triển khai được và chia sẻ được."** 🚀✨