# HelpDesk Project — Interview Questions & Answers

> A mock-interview document. Each section represents a round of questions an interviewer might ask. Answers reference the actual codebase so you can speak confidently.

---

## Round 1 — Project Overview & Motivation

### Q1: Tell me about the project you worked on.

**A:** I worked on a **HelpDesk Management System** — a full-stack web application similar to Stack Overflow. It allows users to register, log in, post questions or issues, comment on them, and vote (upvote/downvote) on content. There is also an admin panel for content moderation, user management (suspend/ban), and viewing reports. The system is built using **Java 17** with **Spring Boot 3.4**, **Spring Security** for authentication/authorization, **Spring Data JPA** with **MySQL** for persistence, and **Thymeleaf** for server-side HTML rendering.

---

### Q2: What problem does this application solve?

**A:** In many organizations and communities, knowledge gets siloed — people ask the same questions repeatedly and answers get lost in emails or chat. This HelpDesk platform centralizes knowledge: users post questions, the community answers through comments, and voting surfaces the best content. Admins can moderate to maintain quality. It's essentially a self-service knowledge base that improves over time.

---

### Q3: What was your specific role and contribution?

**A:** This was a collaborative team project. I worked on the **Comment and Interaction System** — specifically the threaded comments feature (nested replies using a self-referencing `Comment` entity), the voting mechanism (upvote/downvote on both posts and comments), and the content reporting system where users can flag inappropriate content for admin review. I also contributed to the overall architecture decisions and Spring Security configuration.

---

### Q4: How many team members were involved and how did you divide the work?

**A:** We had four team members. We divided work by feature domain:
- **User Authentication & Profiles** — registration, login, profile management, role management
- **Post Management** — CRUD for posts, search functionality, categories, and tags
- **Comments & Interactions** (my part) — threaded comments, voting, reporting
- **Admin Dashboard & Moderation** — admin analytics, content moderation, user suspension/banning

We used a shared **Spring Boot** project with a clean layered architecture (Controller → Service → Repository), so each person could work on their feature independently without major merge conflicts.

---

## Round 2 — Tech Stack & Architecture

### Q5: Why did you choose Spring Boot over other frameworks?

**A:** Several reasons:
1. **Auto-configuration** — Spring Boot reduces boilerplate; we didn't need extensive XML config
2. **Rich ecosystem** — Spring Security, Spring Data JPA, and Thymeleaf integrate seamlessly
3. **Embedded server** — No need to deploy to an external Tomcat; `./mvnw spring-boot:run` starts everything
4. **Production-ready** — Built-in health checks, metrics, and logging support
5. **Industry standard** — Spring is widely used in enterprise Java development, so the skills are transferable

---

### Q6: Walk me through the architecture of your application.

**A:** We followed a **layered MVC architecture**:

1. **Controller Layer** — 10 Spring MVC controllers (e.g., `PostController`, `AdminController`, `VoteController`) handle HTTP requests and return Thymeleaf view names or JSON for REST endpoints.
2. **Service Layer** — 5 service interfaces with 5 implementations (e.g., `PostService` / `PostServiceImpl`). All business logic lives here — validation, orchestration, data transformations between DTOs and entities.
3. **Repository Layer** — 6 Spring Data JPA repositories (e.g., `PostRepository`, `VoteRepository`) extending `JpaRepository`. They provide CRUD operations plus custom queries using `@Query` annotations.
4. **Model Layer** — 6 JPA entities (`User`, `Post`, `Comment`, `Vote`, `Category`, `Role`) mapped to MySQL tables.
5. **View Layer** — Thymeleaf templates for server-side HTML rendering.

The data flows: **Browser → Controller → Service → Repository → MySQL** and back.

---

### Q7: Why Thymeleaf instead of a separate frontend framework like React?

**A:** Thymeleaf was chosen for simplicity and development speed:
- It produces **natural HTML** that works even without a server (designers can preview templates)
- It has **first-class Spring integration** — binding to model attributes, Spring Security tags, form validation
- For a server-rendered application with relatively simple UI interactions, Thymeleaf avoids the complexity of maintaining a separate frontend build pipeline
- That said, if we needed richer client-side interactivity, we'd switch to a **React/Angular SPA** consuming our existing REST endpoints (like the ones in `VoteController` and `CommentRestController`)

---

### Q8: Why MySQL? Did you consider other databases?

**A:** MySQL was chosen because:
- It's a mature, **ACID-compliant** relational database suitable for structured data with relationships (users → posts → comments → votes)
- It has **wide industry adoption** and excellent tooling
- The relational model fits our data naturally — we have many-to-many (users ↔ roles), one-to-many (posts → comments), and self-referencing (comment → replies) relationships

For this project, a NoSQL database like MongoDB wouldn't offer advantages because our data is highly relational. If we needed to scale reads, we could add **Redis caching** without changing the primary database.

---

### Q9: Explain the package structure.

**A:**
```
com.helpdesk
├── config/         — 8 classes: SecurityConfig, custom auth handlers, data initializers, encoder config
├── controller/     — 10 controllers: web (MVC) + REST controllers for AJAX endpoints
├── dto/            — 8 DTOs: decouple API layer from JPA entities
├── model/          — 6 JPA entities: User, Post, Comment, Vote, Category, Role
├── repository/     — 6 interfaces extending JpaRepository with custom queries
└── service/        — 10 files: 5 interfaces + 5 implementations (strategy pattern)
```

This follows the **package-by-layer** convention. Each layer only depends on the layer below it — controllers call services, services call repositories. This enforces **Separation of Concerns**.

---

## Round 3 — Database & JPA

### Q10: Explain your database schema. What are the key entities and relationships?

**A:** We have 6 entities:

| Relationship | Type | Implementation |
|---|---|---|
| User ↔ Role | Many-to-Many | `@ManyToMany` with a join table `users_roles` |
| User → Post | One-to-Many | `@OneToMany(mappedBy = "author")` on User; `@ManyToOne` on Post |
| Post → Comment | One-to-Many | `@OneToMany(mappedBy = "post")` with `orphanRemoval = true` |
| Comment → Comment (replies) | Self-referencing One-to-Many | `parentComment` field with `@ManyToOne`; `replies` with `@OneToMany(mappedBy = "parentComment")` |
| Post → Vote | One-to-Many | `@OneToMany(mappedBy = "post")` |
| Comment → Vote | One-to-Many | `@OneToMany(mappedBy = "comment")` |
| Category → Post | One-to-Many | `@OneToMany(mappedBy = "category")` |

The `Vote` entity can be linked to either a `Post` or a `Comment` (one will be null), enforced by a unique constraint on `(user_id, post_id, comment_id)`.

---

### Q11: How does the self-referencing comment (threaded replies) work?

**A:** The `Comment` entity has two fields:

```java
@ManyToOne
@JoinColumn(name = "parent_id")
private Comment parentComment;

@OneToMany(mappedBy = "parentComment", cascade = CascadeType.ALL)
private List<Comment> replies = new ArrayList<>();
```

- A **top-level comment** has `parentComment = null`
- A **reply** has `parentComment` pointing to the comment it's replying to
- The `replies` list loads all child comments

In the repository, we fetch only top-level comments first:
```java
@Query("SELECT c FROM Comment c WHERE c.post.id = ?1 AND c.parentComment IS NULL ORDER BY c.createdAt ASC")
List<Comment> findTopLevelCommentsByPostId(Long postId);
```

Replies are loaded through the `@OneToMany` relationship. This creates a **tree structure** — which is the **Composite design pattern**.

---

### Q12: What is `ddl-auto=update` and would you use it in production?

**A:** `spring.jpa.hibernate.ddl-auto=update` tells Hibernate to **automatically create or modify** database tables to match the JPA entity annotations. It adds new columns/tables but doesn't drop existing ones.

In **development**, it's convenient because schema changes are applied automatically. In **production**, I would **never** use `update` because:
- It can lead to unexpected schema changes
- It doesn't handle column renames or data migrations
- It doesn't produce auditable migration scripts

In production, I'd use a **migration tool like Flyway or Liquibase** to manage schema changes through versioned SQL scripts.

---

### Q13: What is the `@ElementCollection` annotation you used on the tags field?

**A:** On the `Post` entity, tags are stored as a list of strings:

```java
@ElementCollection
@CollectionTable(name = "post_tags", joinColumns = @JoinColumn(name = "post_id"))
@Column(name = "tag")
private List<String> tags = new ArrayList<>();
```

`@ElementCollection` creates a **separate table** (`post_tags`) with a foreign key to `posts`. Each tag is a simple `String` value, not a full entity. This is simpler than creating a `Tag` entity when tags don't need their own identity or additional fields. The trade-off is that you can't have shared tag metadata across posts.

---

### Q14: Explain the custom `@Query` annotation you used in PostRepository.

**A:** We have a search method that searches across multiple fields:

```java
@Query("SELECT DISTINCT p FROM Post p LEFT JOIN p.tags t LEFT JOIN p.author a WHERE p.hidden = false AND (" +
       "p.title LIKE %?1% OR p.content LIKE %?1% OR p.category.name LIKE %?1% OR t LIKE %?1% OR a.username LIKE %?1%)")
List<Post> searchVisiblePosts(String keyword);
```

This is a **JPQL** (Java Persistence Query Language) query that:
1. Joins the `tags` collection and `author` relationship using `LEFT JOIN`
2. Filters only visible (non-hidden) posts
3. Searches across **title**, **content**, **category name**, **tags**, and **author username**
4. Uses `DISTINCT` to avoid duplicate results from the join
5. Uses positional parameter `?1` bound to the method argument

---

### Q15: How do you handle cascading and orphan removal?

**A:** For example, on the `Post` entity:

```java
@OneToMany(mappedBy = "post", cascade = CascadeType.ALL, orphanRemoval = true)
private List<Vote> votes = new ArrayList<>();
```

- `cascade = CascadeType.ALL` means if I save/delete a `Post`, all its `Vote` children are also saved/deleted
- `orphanRemoval = true` means if I remove a `Vote` from the `votes` list, it gets deleted from the database

This ensures **referential integrity** — when a post is deleted, all its votes and comments are automatically cleaned up.

---

## Round 4 — Spring Security

### Q16: How does authentication work in your application?

**A:** We use **Spring Security** with a multi-step custom authentication flow:

1. User submits username and password on the login form
2. **`CustomAuthenticationProvider`** (extends `DaoAuthenticationProvider`) intercepts the request
3. It first delegates to the parent class to validate credentials against the database (using our `UserServiceImpl` which implements `UserDetailsService`)
4. Passwords are compared using **BCrypt** hashing
5. If credentials are valid, it checks the **account status**:
   - `BANNED` → throws `LockedException` with the ban reason
   - `SUSPENDED` → checks if the suspension period has expired. If not, throws `LockedException`. If expired, **auto-reactivates** the account
   - `ACTIVE` → proceeds with authentication
6. On success, **`CustomAuthenticationSuccessHandler`** updates the `lastLoginDate` field
7. On failure, **`CustomAuthenticationFailureHandler`** redirects to `/login?error=` with the specific error message

---

### Q17: How do you implement role-based access control?

**A:** Two levels:

**1. URL-level security** in `SecurityConfig`:
```java
.requestMatchers("/admin/**").hasRole("ADMIN")
.requestMatchers("/posts/**", "/categories/**").authenticated()
.requestMatchers("/registration/**").permitAll()
```

**2. Method-level security** with `@PreAuthorize`:
```java
@PreAuthorize("hasRole('ADMIN')")
public String adminDashboard(Model model) { ... }
```

We enable this with `@EnableMethodSecurity(securedEnabled = true, prePostEnabled = true)` on `SecurityConfig`. Roles are stored in the `roles` table and loaded via the `User` entity's `@ManyToMany` relationship with `Role`.

---

### Q18: How is the password stored and validated?

**A:** We use **BCrypt**, a one-way adaptive hashing algorithm:

- **Storage**: During registration, the password is hashed using `BCryptPasswordEncoder.encode(rawPassword)` before saving to the database. The actual password is never stored.
- **Validation**: During login, BCrypt compares the submitted password against the stored hash using `BCryptPasswordEncoder.matches(rawPassword, encodedPassword)`. This is handled internally by Spring Security's `DaoAuthenticationProvider`.
- **Why BCrypt?** It includes a built-in salt (preventing rainbow table attacks) and a configurable work factor that makes brute-force attacks computationally expensive.

---

### Q19: How does the suspended account auto-reactivation work?

**A:** In `CustomAuthenticationProvider.authenticate()`:

```java
if ("SUSPENDED".equals(user.getAccountStatus())) {
    if (user.getSuspensionEndDate() != null && user.getSuspensionEndDate().after(new Date())) {
        // Suspension still active — deny login
        throw new LockedException("Your account is suspended until " + 
            dateFormat.format(user.getSuspensionEndDate()) + ". Reason: " + user.getSuspensionReason());
    } else {
        // Suspension expired — automatically reactivate
        user.setAccountStatus("ACTIVE");
        user.setSuspensionReason(null);
        user.setSuspensionEndDate(null);
        userRepository.save(user);
    }
}
```

If the suspension end date has passed, the system **automatically clears** the suspension and allows login. This means admins don't need to manually reactivate users after temporary suspensions.

---

### Q20: CSRF protection is disabled in your SecurityConfig. Is that a concern?

**A:** Yes, disabling CSRF is a **security trade-off** made for development convenience. In `SecurityConfig`:
```java
.csrf(csrf -> csrf.disable())
```

This creates a vulnerability where a malicious site could make authenticated requests on behalf of a logged-in user. We disabled it to simplify our AJAX calls to REST endpoints (`/api/votes/`, `/api/comments/`), but this is **not recommended for production**. The proper approach would be to:
1. **Keep CSRF enabled** (the default in Spring Security 6)
2. Include a CSRF meta tag in the Thymeleaf layout: `<meta name="_csrf" th:content="${_csrf.token}"/>`
3. Send the token as an HTTP header in AJAX requests: `headers: { 'X-CSRF-TOKEN': csrfToken }`

Spring Security has built-in support for CSRF with AJAX, so disabling it entirely is unnecessary.

---

## Round 5 — Design Patterns

### Q21: What design patterns did you use? Give specific examples.

**A:**

| Pattern | Example |
|---|---|
| **MVC** | `PostController` (Controller) → `PostServiceImpl` (Model/Business Logic) → `list.html` (View) |
| **Repository** | `PostRepository extends JpaRepository<Post, Long>` — abstracts all SQL behind method names |
| **DTO** | `PostDto` decouples the `Post` JPA entity from the web layer. Controllers only deal with DTOs, never entities directly |
| **Service Layer** | `PostService` interface + `PostServiceImpl` — all business logic (validation, DTO conversion, authorization checks) is here, not in controllers |
| **Dependency Injection** | Constructor injection throughout: `public PostController(PostService postService, CategoryService categoryService)` |
| **Strategy** | `PostService` is an interface — we could swap `PostServiceImpl` for a different implementation without changing any controller |
| **Composite** | `Comment` entity with `parentComment` and `replies` forms a tree structure |
| **Observer** | `AuthenticationSuccessListener` listens for Spring Security's `AuthenticationSuccessEvent` to update last login date |
| **Singleton** | All Spring beans (`@Service`, `@Repository`, `@Controller`) are singletons by default |
| **Proxy** | Spring creates AOP proxies for `@Transactional` methods and `@PreAuthorize` security checks |
| **Template Method** | `CustomAuthenticationProvider` extends `DaoAuthenticationProvider` and overrides `authenticate()` — the parent class defines the template, we customize one step |

---

### Q22: Explain the DTO pattern. Why not just return entities directly?

**A:** DTOs (Data Transfer Objects) are plain Java objects that carry data between layers. For example, `PostDto` has fields like `title`, `content`, `categoryId`, `tags`, `authorUsername`, `voteScore` — but it does NOT have JPA annotations, lazy-loaded relationships, or circular references.

Returning entities directly causes problems:
1. **Circular references** — `Post` has a `User` author, `User` has a list of `Posts` → infinite loop during JSON serialization
2. **Lazy loading exceptions** — accessing `post.getComments()` outside a Hibernate session throws `LazyInitializationException`
3. **Over-exposure** — the `User` entity contains the `password` hash; returning it to the frontend is a security risk
4. **Coupling** — if the database schema changes, the API contract shouldn't need to change

So in the service layer, we convert: `Entity → DTO` (outgoing) and `DTO → Entity` (incoming).

---

### Q23: How is the Observer pattern used in your project?

**A:** We have `AuthenticationSuccessListener`:

```java
@Component
public class AuthenticationSuccessListener implements ApplicationListener<AuthenticationSuccessEvent> {
    @Override
    @Transactional
    public void onApplicationEvent(AuthenticationSuccessEvent event) {
        Object principal = event.getAuthentication().getPrincipal();
        if (principal instanceof UserDetails) {
            String username = ((UserDetails) principal).getUsername();
            userService.updateLastLoginDate(username);
        }
    }
}
```

Spring Security **publishes** an `AuthenticationSuccessEvent` whenever a user logs in successfully. Our listener **observes** that event and updates the user's `lastLoginDate`. This decouples the login audit logic from the authentication logic — the authentication system doesn't need to know about the audit, and we could add more listeners (e.g., logging, notifications) without modifying the authentication code.

---

## Round 6 — Code-Specific Deep Dives

### Q24: How does the voting system work? Walk me through the code.

**A:** The voting system allows users to upvote or downvote posts and comments.

**Controller** (`VoteController`):
```java
@PostMapping("/post/{postId}/upvote")
public ResponseEntity<VoteDto> upvotePost(@PathVariable Long postId, Authentication authentication) {
    return ResponseEntity.ok(voteService.votePost(postId, true, authentication.getName()));
}
```

**Service** (`VoteServiceImpl`): The key logic is the **toggle behavior**:
1. Look up if the user already voted on this post: `voteRepository.findByUserAndPost(user, post)`
2. If **no existing vote** → create a new `Vote` with `upvote = true/false`
3. If **same vote exists** (user upvoted and clicks upvote again) → **remove** the vote (toggle off)
4. If **opposite vote exists** (user upvoted and clicks downvote) → **change** the vote direction

**Entity** (`Post`): Vote score is calculated in the entity:
```java
public int getVoteScore() {
    return getUpvoteCount() - getDownvoteCount();  // upvotes minus downvotes
}
```

**Database**: A `@UniqueConstraint(columnNames = {"user_id", "post_id", "comment_id"})` ensures one vote per user per item.

---

### Q25: How does the reporting and moderation system work?

**A:** The moderation flow has three stages:

**1. User reports content** — `ReportController` handles `POST /api/reports/post/{id}` and `POST /api/reports/comment/{id}`. This sets `reported = true`, `reportReason`, `reportedBy`, and `reportedAt` on the entity.

**2. Admin reviews reports** — `AdminController` at `GET /admin/reports` fetches all reported posts and comments:
```java
List<PostDto> reportedPosts = postService.getReportedPosts();
List<CommentDto> reportedComments = commentService.getReportedComments();
```
These are shown on the admin dashboard.

**3. Admin takes action** — The admin can:
- **Hide** content (soft-delete): `POST /admin/posts/{id}/hide` sets `hidden = true` without deleting the database row
- **Delete** content: `POST /admin/posts/{id}/delete` permanently removes it
- **Resolve** the report: `POST /admin/reports/post/{id}/resolve` clears the `reported` flag
- **Suspend/Ban** the user: `POST /admin/users/{id}/toggle-status` or `POST /admin/users/{id}/ban`

Hidden posts are filtered out in queries: `findByHiddenFalse()`, `findByCategoryAndHiddenFalse()`, etc. — so regular users never see them, but admins can still view and manage them.

---

### Q26: How does the search functionality work?

**A:** Search is handled through a custom JPQL query in `PostRepository`:

```java
@Query("SELECT DISTINCT p FROM Post p LEFT JOIN p.tags t LEFT JOIN p.author a WHERE p.hidden = false AND (" +
       "p.title LIKE %?1% OR p.content LIKE %?1% OR p.category.name LIKE %?1% OR t LIKE %?1% OR a.username LIKE %?1%)")
List<Post> searchVisiblePosts(String keyword);
```

This searches across **five fields** simultaneously: post title, content, category name, tags, and author username. The `LEFT JOIN` ensures posts without tags or an author still appear in results. `DISTINCT` prevents duplicate rows from the join. Only non-hidden posts are returned.

The `PostController` exposes it at `GET /posts/search?query=keyword`:
```java
@GetMapping("/search")
public String searchPosts(@RequestParam("query") String query, Model model) {
    model.addAttribute("posts", postService.searchPosts(query));
    return "posts/search-results";
}
```

---

### Q27: Explain the `DataInitializer`. Why do you need it?

**A:** `DataInitializer` implements `CommandLineRunner`, which means it runs **on application startup**:

```java
@Override
public void run(String... args) {
    if (userRepository.findByUsername("admin") == null) {
        User adminUser = new User();
        adminUser.setUsername("admin");
        adminUser.setPassword(passwordEncoder.encode("admin123"));
        adminUser.setRoles(Arrays.asList(new Role("ROLE_ADMIN"), new Role("ROLE_USER")));
        userRepository.save(adminUser);
    }
}
```

It creates a **default admin user** so there's always someone who can access the admin dashboard. Without it, after a fresh database setup, no one could access admin features to create the first admin account. The `if` check ensures it doesn't create duplicates on subsequent startups.

We also have a `CategoryDataInitializer` (with `@Order(2)`) that creates 6 default categories (Technical Support, Software Development, etc.) so the application has usable categories immediately.

---

### Q28: How do you handle the `@PrePersist` and `@PreUpdate` lifecycle callbacks?

**A:** On every entity, we have:

```java
@PrePersist
protected void onCreate() {
    createdAt = new Date();
    updatedAt = new Date();
}

@PreUpdate
protected void onUpdate() {
    updatedAt = new Date();
}
```

These are **JPA lifecycle callbacks**:
- `@PrePersist` fires just before an entity is first inserted — we set both timestamps
- `@PreUpdate` fires before any subsequent update — we refresh `updatedAt`

This ensures timestamps are always accurate without the service layer needing to set them manually. It's a cross-cutting concern handled at the entity level.

---

## Round 7 — Scenario & Problem-Solving Questions

### Q29: What would happen if two users vote on the same post at the same time?

**A:** The `@UniqueConstraint(columnNames = {"user_id", "post_id", "comment_id"})` on the `Vote` table prevents duplicate votes at the database level. If a race condition caused two simultaneous vote requests for the same user on the same post, the second `save()` would throw a `DataIntegrityViolationException`. The current code doesn't explicitly handle this exception with a try-catch, so it would return a 500 error. To improve this, I'd wrap the vote logic in a `@Transactional` method with a `try-catch` that gracefully handles the constraint violation.

---

### Q30: How would you add pagination to the posts list?

**A:** Spring Data JPA makes this easy. I'd change the repository method signature:

```java
// Before
List<Post> findByHiddenFalse();

// After
Page<Post> findByHiddenFalse(Pageable pageable);
```

In the service:
```java
public Page<PostDto> getAllPosts(int page, int size) {
    return postRepository.findByHiddenFalse(PageRequest.of(page, size, Sort.by("createdAt").descending()));
}
```

In the controller, pass `page` and `size` as query parameters, and in the Thymeleaf template, render page navigation links using the `Page` object's `totalPages`, `number`, `hasNext()`, and `hasPrevious()` methods.

---

### Q31: How would you add email verification during registration?

**A:** I'd:
1. Add a `verified` boolean and `verificationToken` field to the `User` entity
2. On registration, generate a UUID token, save it, and send an email with a link like `/verify?token=abc123`
3. Create a `GET /verify` endpoint that looks up the token, sets `verified = true`, and clears the token
4. In `CustomAuthenticationProvider`, add a check: if the user isn't verified, throw an exception
5. Use **Spring Mail** (`spring-boot-starter-mail`) to send the verification email via SMTP

---

### Q32: The application uses `ddl-auto=update`. What problems could this cause?

**A:** Several:
1. **Renaming a column** — Hibernate creates a new column instead of renaming; the old column with data remains
2. **Changing a column type** — may fail or cause data loss
3. **Removing a field** — the database column stays; Hibernate never drops columns
4. **No rollback** — if a schema change fails midway, you can't roll back
5. **No audit trail** — no record of what changed and when
6. **Team coordination** — different developers' entity changes can conflict

The fix: use **Flyway** or **Liquibase** with versioned migration scripts (`V1__create_users.sql`, `V2__add_account_status.sql`) that are applied in order, can be reviewed, and support rollback.

---

### Q33: How would you improve the search to handle large datasets?

**A:** The current `LIKE %keyword%` search doesn't use database indexes (the leading wildcard prevents it). For better performance:
1. **Short term**: Add **full-text indexes** in MySQL and use `MATCH ... AGAINST` syntax via native queries
2. **Medium term**: Integrate **Elasticsearch** — index posts asynchronously, query Elasticsearch for search, and use MySQL as the source of truth
3. **Caching**: Cache popular search results in **Redis** with TTL
4. **Pagination**: Always paginate results (never return all matches at once)

---

### Q34: How would you scale this application for 100,000 concurrent users?

**A:** I'd address multiple layers:

**Application layer**:
- Deploy **multiple instances** behind a **load balancer** (Nginx / AWS ALB)
- Use **stateless sessions** (store sessions in Redis) so any instance can serve any request
- Extract REST APIs and potentially move to a **React frontend + API backend** architecture

**Database layer**:
- **Read replicas** for MySQL to distribute read queries
- **Connection pooling** — Spring Boot already uses HikariCP
- **Caching** — Redis for frequently accessed data (post lists, vote counts)

**Infrastructure**:
- **Containerize** with Docker, orchestrate with Kubernetes
- Use **CDN** for static assets (CSS, JavaScript)
- Add **message queues** (RabbitMQ/Kafka) for asynchronous operations (email notifications, vote count updates)

---

### Q35: What would you change if you were to rebuild this from scratch?

**A:**
1. **Frontend**: Use React or Angular instead of Thymeleaf for a richer user experience
2. **API design**: Build a pure REST API backend with OpenAPI/Swagger documentation
3. **Database migrations**: Use Flyway from day one
4. **Testing**: Add comprehensive unit tests (JUnit + Mockito) for services and integration tests for controllers
5. **Caching**: Add Redis caching for vote counts and popular posts
6. **Search**: Use Elasticsearch instead of `LIKE` queries
7. **Notifications**: Add real-time notifications using WebSockets when someone replies to your post
8. **CI/CD**: Set up GitHub Actions for automated testing and deployment

---

## Round 8 — Spring-Specific Technical Questions

### Q36: What is the difference between `@Controller` and `@RestController`?

**A:** In our project, we use both:
- `@Controller` (e.g., `PostController`) — returns a **view name** (like `"posts/list"`), which Thymeleaf resolves to an HTML template
- `@RestController` (e.g., `VoteController`, `CommentRestController`) — returns **data directly** (JSON), which is what our JavaScript AJAX calls expect

`@RestController` is equivalent to `@Controller + @ResponseBody` on every method. We use `@Controller` for page navigation and `@RestController` for API endpoints that the frontend consumes via `fetch()`.

---

### Q37: Explain `@Transactional` and where you use it.

**A:** `@Transactional` ensures that a method runs within a **database transaction** — if any step fails, all changes are rolled back. We use it in:

- `CustomAuthenticationSuccessHandler.onAuthenticationSuccess()` — updating `lastLoginDate` must be atomic
- `AuthenticationSuccessListener.onApplicationEvent()` — same reason
- Service methods that perform multiple repository calls (e.g., creating a post and saving tags together)

Spring creates an **AOP proxy** around `@Transactional` methods. When the method starts, a transaction begins. If it completes normally, the transaction commits. If it throws a runtime exception, the transaction rolls back.

---

### Q38: What is `FetchType.EAGER` vs `FetchType.LAZY`? Where do you use each?

**A:** 
- `EAGER` — loads the related data immediately when the parent entity is loaded
- `LAZY` — loads the related data only when it's accessed (deferred loading)

In our project:
- `User.roles` uses `FetchType.EAGER` — because roles are **always needed** during authentication (Spring Security checks roles on every request)
- `Post.comments`, `Post.votes` use the default `LAZY` — because we don't always need all comments and votes when listing posts

Using `EAGER` everywhere would cause **N+1 problems** — loading a list of 100 posts would trigger 100 extra queries for comments and 100 more for votes. `LAZY` loading avoids this.

---

### Q39: What is the N+1 problem? Does your application have it?

**A:** The N+1 problem occurs when loading a list of N entities triggers N additional queries for related data. For example, loading 50 posts with `EAGER` fetching on comments would execute: 1 query for posts + 50 queries for comments = 51 queries.

Our application **partially** has this issue. When we load a single post with its comments and votes, Hibernate issues separate queries for each relationship. To fix this:
1. Use **`JOIN FETCH`** in JPQL: `SELECT p FROM Post p JOIN FETCH p.comments WHERE p.id = ?1`
2. Use **`@EntityGraph`** to define which relationships to load eagerly for specific queries
3. Use **batch fetching**: `@BatchSize(size = 25)` on collections

---

### Q40: How does Spring dependency injection work in your project?

**A:** We use **constructor injection** throughout. For example:

```java
@Controller
public class PostController {
    private final PostService postService;
    private final CategoryService categoryService;
    
    @Autowired
    public PostController(PostService postService, CategoryService categoryService) {
        this.postService = postService;
        this.categoryService = categoryService;
    }
}
```

Spring's IoC container:
1. Scans for `@Component`, `@Service`, `@Repository`, `@Controller` annotations
2. Creates singleton instances of each (the bean)
3. Resolves constructor dependencies — `PostController` needs a `PostService`, so Spring injects `PostServiceImpl` (the only implementation)
4. Fields are `final`, making them immutable after construction

Constructor injection is preferred over `@Autowired` on fields because:
- Dependencies are **explicit** and **required** (can't create the object without them)
- Fields can be `final` (thread-safe)
- Easier to write unit tests (pass mock dependencies through the constructor)

---

## Round 9 — Testing & Quality

### Q41: How would you test the voting service?

**A:** I'd write unit tests with **JUnit 5 + Mockito**:

```java
@ExtendWith(MockitoExtension.class)
class VoteServiceImplTest {
    @Mock private VoteRepository voteRepository;
    @Mock private PostRepository postRepository;
    @Mock private UserRepository userRepository;
    @InjectMocks private VoteServiceImpl voteService;
    
    @Test
    void shouldCreateNewUpvoteWhenNoExistingVote() {
        // Arrange: mock user, post, and empty vote lookup
        when(userRepository.findByUsername("john")).thenReturn(mockUser);
        when(postRepository.findById(1L)).thenReturn(Optional.of(mockPost));
        when(voteRepository.findByUserAndPost(mockUser, mockPost)).thenReturn(Optional.empty());
        
        // Act
        VoteDto result = voteService.votePost(1L, true, "john");
        
        // Assert
        verify(voteRepository).save(any(Vote.class));
        assertTrue(result.isUpvoted());
    }
    
    @Test
    void shouldToggleOffWhenVotingSameDirection() { ... }
    
    @Test
    void shouldSwitchDirectionWhenVotingOpposite() { ... }
}
```

For integration tests, I'd use `@SpringBootTest` with an **H2 in-memory database** to test the full stack from controller to database.

---

### Q42: What would you log and monitor in production?

**A:** Key observability concerns:
1. **Login attempts** — successful and failed (we already track `lastLoginDate`; I'd add a failed attempt counter)
2. **API response times** — Spring Boot Actuator + Micrometer for metrics
3. **Error rates** — 4xx and 5xx responses per endpoint
4. **Database query performance** — slow query log in MySQL + Hibernate statistics
5. **Report volume** — spike in reports could indicate a spam attack
6. **Active users** — sessions count, concurrent users

I'd use **SLF4J + Logback** (already included in Spring Boot), **Spring Boot Actuator** for health/metrics endpoints, and export metrics to **Prometheus/Grafana** for dashboards and alerts.

---

## Round 10 — Behavioral & Soft-Skill Questions

### Q43: What was the most challenging part of this project?

**A:** The **threaded comments system** was the most challenging. Implementing a self-referencing entity where a `Comment` can have child `Comment` objects required careful handling of:
- Recursive data loading (avoiding infinite loops with `@JsonIgnore` or DTOs)
- Cascading deletes (deleting a parent comment should delete all its replies)
- Ordering (top-level comments by newest first, replies by oldest first)
- The UI rendering — displaying nested comments at different indentation levels using Thymeleaf recursive fragments

The **voting toggle logic** was also tricky — handling the three states (new vote, toggle off, switch direction) in a clean, transaction-safe way.

---

### Q44: If a non-technical stakeholder asked you to explain this project, what would you say?

**A:** "Imagine a company's internal Q&A board — like a simplified version of Stack Overflow. Employees post questions, others answer in the comments, and everyone can vote on which answers are most helpful. The best answers rise to the top. There's also a team of moderators who can remove inappropriate content and manage user accounts. It helps the organization capture and share knowledge instead of answering the same questions over and over."

---

### Q45: How did your team coordinate during development?

**A:** We used:
- **Git/GitHub** for version control with feature branches
- **Package-by-layer** architecture so each person worked in their own domain (posts, comments, admin, auth)
- Clear **interfaces** between layers — we agreed on service interface signatures early, so one person could build the controller while another built the service implementation
- Regular code reviews to maintain consistency

The layered architecture and interface-based design were crucial — they minimized merge conflicts and allowed parallel development.
