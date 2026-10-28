# פתרונות לתרגילים – Spring Security + JWT

כל הקוד מבוסס על הפרויקט מהשיעור (חבילה `com.example.jwtlesson`). לכל תרגיל מסומן מה **חדש** ומה **משתנה**.

## סיכום קבצים

| קובץ | סטטוס | תרגיל |
|---|---|---|
| `exception/UserNotFoundException.java` | חדש | 3, 5 |
| `exception/BusinessRuleException.java` | חדש | 4, 5 |
| `exception/InvalidRefreshTokenException.java` | חדש | 7 |
| `exception/GlobalExceptionHandler.java` | **מוחלף** | 3, 4, 5, 7 |
| `dto/AddRoleRequest.java` | חדש | 3 |
| `dto/SetEnabledRequest.java` | חדש | 5 |
| `dto/ChangePasswordRequest.java` | חדש | 4 |
| `dto/RefreshRequest.java` | חדש | 7 |
| `dto/LoginResponse.java` | **מוחלף** | 7 |
| `service/AdminUserService.java` | חדש | 1, 3, 5, 6 |
| `service/UserService.java` | **מוחלף** | 4, 6 |
| `service/UserLoginService.java` | **מוחלף** | 7 |
| `service/RefreshTokenService.java` | חדש | 7 |
| `entity/RefreshToken.java` | חדש | 7 |
| `repository/RefreshTokenRepository.java` | חדש | 7 |
| `controller/AdminController.java` | חדש | 1, 3, 5 |
| `controller/AccountController.java` | חדש | 4 |
| `controller/UserLoginController.java` | **מוחלף** | 7 |
| `config/SecurityConfig.java` | שורה אחת | 6 |
| `application.properties` | שינוי | 2, 7 |
| `application-short.properties` | חדש | 2 |
| `pom.xml` | תלות אחת | 2 |
| `src/test/.../JwtUtilExpiryTest.java` | חדש | 2 |
| `http/exercises-tests.http` | חדש | הכול |

---

## קבצים משותפים (נדרשים לכמה תרגילים)

### `exception/UserNotFoundException.java`
```java
package com.example.jwtlesson.exception;

public class UserNotFoundException extends RuntimeException {

    public UserNotFoundException(Long id) {
        super("User not found: id=" + id);
    }
}
```

### `exception/BusinessRuleException.java`
```java
package com.example.jwtlesson.exception;

/** A request that is well-formed but breaks a business rule -> 400. */
public class BusinessRuleException extends RuntimeException {

    public BusinessRuleException(String message) {
        super(message);
    }
}
```

### `exception/InvalidRefreshTokenException.java`
```java
package com.example.jwtlesson.exception;

public class InvalidRefreshTokenException extends RuntimeException {

    public InvalidRefreshTokenException() {
        super("Invalid or expired refresh token");
    }
}
```

### `exception/GlobalExceptionHandler.java` (מחליף את הקיים)
```java
package com.example.jwtlesson.exception;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.http.converter.HttpMessageNotReadableException;
import org.springframework.security.core.AuthenticationException;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.util.LinkedHashMap;
import java.util.Map;

@RestControllerAdvice
public class GlobalExceptionHandler {

    // Wrong password, unknown user, disabled user -> the same answer (don't reveal which)
    @ExceptionHandler(AuthenticationException.class)
    public ResponseEntity<Map<String, Object>> handleAuthentication(AuthenticationException ex) {
        return error(HttpStatus.UNAUTHORIZED, "Invalid username or password");
    }

    @ExceptionHandler(InvalidRefreshTokenException.class)
    public ResponseEntity<Map<String, Object>> handleInvalidRefresh(InvalidRefreshTokenException ex) {
        return error(HttpStatus.UNAUTHORIZED, ex.getMessage());
    }

    @ExceptionHandler(UsernameTakenException.class)
    public ResponseEntity<Map<String, Object>> handleUsernameTaken(UsernameTakenException ex) {
        return error(HttpStatus.CONFLICT, ex.getMessage());
    }

    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<Map<String, Object>> handleUserNotFound(UserNotFoundException ex) {
        return error(HttpStatus.NOT_FOUND, ex.getMessage());
    }

    @ExceptionHandler(BusinessRuleException.class)
    public ResponseEntity<Map<String, Object>> handleBusinessRule(BusinessRuleException ex) {
        return error(HttpStatus.BAD_REQUEST, ex.getMessage());
    }

    // Broken JSON, or a value that is not a valid enum constant (e.g. "role": "ROLE_SUPERMAN")
    @ExceptionHandler(HttpMessageNotReadableException.class)
    public ResponseEntity<Map<String, Object>> handleUnreadable(HttpMessageNotReadableException ex) {
        return error(HttpStatus.BAD_REQUEST, "Malformed JSON or invalid value");
    }

    // @Valid failed: {"status":400,"error":"Validation failed","fields":{"password":"size must be ..."}}
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, Object>> handleValidation(MethodArgumentNotValidException ex) {
        Map<String, String> fields = new LinkedHashMap<>();
        ex.getBindingResult().getFieldErrors()
                .forEach(fe -> fields.put(fe.getField(), fe.getDefaultMessage()));

        Map<String, Object> body = new LinkedHashMap<>();
        body.put("status", HttpStatus.BAD_REQUEST.value());
        body.put("error", "Validation failed");
        body.put("fields", fields);
        return ResponseEntity.badRequest().body(body);
    }

    private ResponseEntity<Map<String, Object>> error(HttpStatus status, String message) {
        Map<String, Object> body = new LinkedHashMap<>();
        body.put("status", status.value());
        body.put("error", message);
        return ResponseEntity.status(status).body(body);
    }
}
```

---

## תרגיל 1 (קל) – `GET /api/admin/stats`

החוק `/api/admin/**` → `hasRole("ADMIN")` כבר קיים ב‑`SecurityConfig`, אז **לא צריך לשנות שום חוק אבטחה**. רק מוסיפים Service ו‑Controller.

### `service/AdminUserService.java`
(הגרסה המלאה, שמשמשת גם את תרגילים 3, 5 ו‑6)
```java
package com.example.jwtlesson.service;

import com.example.jwtlesson.dto.UserResponse;
import com.example.jwtlesson.entity.Role;
import com.example.jwtlesson.entity.RoleName;
import com.example.jwtlesson.entity.User;
import com.example.jwtlesson.exception.BusinessRuleException;
import com.example.jwtlesson.exception.UserNotFoundException;
import com.example.jwtlesson.repository.RoleRepository;
import com.example.jwtlesson.repository.UserRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

/**
 * Admin-only operations.
 * Two layers of protection: the URL rule in SecurityConfig AND @PreAuthorize here (exercise 6).
 */
@Service
@RequiredArgsConstructor
@PreAuthorize("hasRole('ADMIN')")
public class AdminUserService {

    private final UserRepository userRepository;
    private final RoleRepository roleRepository;

    // ---------- Exercise 1 ----------
    @Transactional(readOnly = true)
    public long countUsers() {
        return userRepository.count();
    }

    // ---------- Exercise 3 ----------
    @Transactional
    public UserResponse addRole(Long userId, RoleName roleName) {
        User user = userRepository.findById(userId)
                .orElseThrow(() -> new UserNotFoundException(userId));

        Role role = roleRepository.findByName(roleName)
                .orElseThrow(() -> new IllegalStateException("Role missing in DB: " + roleName));

        // roles is a Set, so adding an existing role is harmless (idempotent).
        // The user is a managed entity: Hibernate writes user_roles when the transaction commits.
        user.getRoles().add(role);

        return UserResponse.from(user);
    }

    // ---------- Exercise 5 ----------
    @Transactional
    public UserResponse setEnabled(Long userId, boolean enabled, String currentAdminUsername) {
        User user = userRepository.findById(userId)
                .orElseThrow(() -> new UserNotFoundException(userId));

        // Protect against an admin locking himself out
        if (!enabled && user.getUsername().equals(currentAdminUsername)) {
            throw new BusinessRuleException("You cannot disable your own account");
        }

        user.setEnabled(enabled);
        return UserResponse.from(user);
    }
}
```

### `controller/AdminController.java`
(מכיל גם את נקודות הקצה של תרגילים 3 ו‑5)
```java
package com.example.jwtlesson.controller;

import com.example.jwtlesson.dto.AddRoleRequest;
import com.example.jwtlesson.dto.SetEnabledRequest;
import com.example.jwtlesson.dto.UserResponse;
import com.example.jwtlesson.service.AdminUserService;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.security.core.Authentication;
import org.springframework.web.bind.annotation.*;

import java.util.Map;

/** All URLs here are under /api/admin/** -> SecurityConfig requires ROLE_ADMIN. */
@RestController
@RequestMapping("/api/admin")
@RequiredArgsConstructor
public class AdminController {

    private final AdminUserService adminUserService;

    // Exercise 1: GET /api/admin/stats -> {"totalUsers": 3}
    @GetMapping("/stats")
    public Map<String, Long> stats() {
        return Map.of("totalUsers", adminUserService.countUsers());
    }

    // Exercise 3: PUT /api/admin/users/{id}/roles   {"role":"ROLE_ADMIN"}
    @PutMapping("/users/{id}/roles")
    public UserResponse addRole(@PathVariable Long id,
                                @Valid @RequestBody AddRoleRequest request) {
        return adminUserService.addRole(id, request.role());
    }

    // Exercise 5: PUT /api/admin/users/{id}/enabled   {"enabled":false}
    @PutMapping("/users/{id}/enabled")
    public UserResponse setEnabled(@PathVariable Long id,
                                   @Valid @RequestBody SetEnabledRequest request,
                                   Authentication authentication) {
        return adminUserService.setEnabled(id, request.enabled(), authentication.getName());
    }
}
```

---

## תרגיל 2 (קל) – תוקף טוקן של 60 שניות

שלוש דרכים, מהפשוטה לאוטומטית.

### א. פרופיל Spring ייעודי – `src/main/resources/application-short.properties`
```properties
# Activated with: --spring.profiles.active=short   (IntelliJ: Run Configuration -> Active profiles = short)
jwt.expiration-ms=60000
```
נקודת הכניסה לא משתנה, ובסיום פשוט מריצים בלי הפרופיל. אין צורך לזכור להחזיר ל‑`3600000`.

### ב. הבדיקה הידנית (בדיקה 20 בקובץ השיעור)
1. מריצים את השרת עם הפרופיל `short`.
2. מריצים בקשה 7 (לוגין של `user`).
3. ממתינים יותר מדקה, ואז מריצים בקשה 8 ← מצפים ל‑**401**.

### ג. בדיקת יחידה אוטומטית

הוסיפו ל‑`pom.xml`:
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

`src/test/java/com/example/jwtlesson/security/JwtUtilExpiryTest.java`
```java
package com.example.jwtlesson.security;

import io.jsonwebtoken.ExpiredJwtException;
import io.jsonwebtoken.JwtException;
import org.junit.jupiter.api.Test;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetails;

import static org.assertj.core.api.Assertions.assertThat;
import static org.junit.jupiter.api.Assertions.assertThrows;

/** Plain unit test - no Spring context, no database. */
class JwtUtilExpiryTest {

    private static final String SECRET = "47o9ZwPSwJH0HIGIpS8wuGOJOLfiN40HPSvI/k+kv88=";

    private final UserDetails user = User.withUsername("user")
            .password("irrelevant")
            .authorities("ROLE_USER")
            .build();

    @Test
    void validTokenIsAccepted() {
        JwtUtil jwtUtil = new JwtUtil(SECRET, 60_000);

        String token = jwtUtil.generateToken(user);

        assertThat(jwtUtil.extractUsername(token)).isEqualTo("user");
    }

    @Test
    void expiredTokenIsRejected() throws InterruptedException {
        JwtUtil jwtUtil = new JwtUtil(SECRET, 1_000); // 1 second

        String token = jwtUtil.generateToken(user);
        Thread.sleep(2_100); // "exp" has one-second precision, so wait a bit more than 2 seconds

        assertThrows(ExpiredJwtException.class, () -> jwtUtil.parseClaims(token));
    }

    @Test
    void tamperedTokenIsRejected() {
        JwtUtil jwtUtil = new JwtUtil(SECRET, 60_000);
        String token = jwtUtil.generateToken(user);

        // Flip the last character of the signature
        char last = token.charAt(token.length() - 1);
        String tampered = token.substring(0, token.length() - 1) + (last == 'A' ? 'B' : 'A');

        assertThrows(JwtException.class, () -> jwtUtil.parseClaims(tampered));
    }
}
```

---

## תרגיל 3 (בינוני) – `PUT /api/admin/users/{id}/roles`

מוסיפים רק DTO. הלוגיקה ב‑`AdminUserService.addRole` והנקודה ב‑`AdminController` (למעלה).

### `dto/AddRoleRequest.java`
```java
package com.example.jwtlesson.dto;

import com.example.jwtlesson.entity.RoleName;
import jakarta.validation.constraints.NotNull;

public record AddRoleRequest(@NotNull RoleName role) {
}
```

### למה ה‑user מקבל גישה בלי לוגין מחדש?
ה‑JWT מכיל claim בשם `roles`, אבל **השרת לא סומך עליו**. בכל בקשה `JwtAuthFilter` מוסיף ל‑`SecurityContext` את התפקידים לפי מה שנטען עכשיו מה‑DB דרך `loadUserByUsername`. הטוקן משמש רק להוכחת זהות (`sub` + חתימה + תוקף). אחרי שהמנהל שינה את `user_roles`, הבקשה הבאה כבר רואה את התפקיד החדש.
המחיר: שאילתה לכל בקשה. היתרון: הרשאות מתעדכנות מיד.
ה‑claim `roles` בטוקן הישן ישאר ישן, ולכן לקוח שמסתמך עליו להצגת כפתורים יראה מידע לא מעודכן עד לוגין חדש.

---

## תרגיל 4 (בינוני) – `PUT /api/user/password`

### `dto/ChangePasswordRequest.java`
```java
package com.example.jwtlesson.dto;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;

public record ChangePasswordRequest(
        @NotBlank String oldPassword,
        @NotBlank @Size(min = 8, max = 72) String newPassword) { // BCrypt uses at most 72 bytes
}
```

### `service/UserService.java` (מחליף את הקיים; כולל גם את תרגיל 6)
```java
package com.example.jwtlesson.service;

import com.example.jwtlesson.dto.UserResponse;
import com.example.jwtlesson.entity.User;
import com.example.jwtlesson.exception.BusinessRuleException;
import com.example.jwtlesson.repository.UserRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
@RequiredArgsConstructor
public class UserService {

    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;

    // Exercise 6: an admin may read anyone, a regular user only himself
    @PreAuthorize("hasRole('ADMIN') or #username == authentication.name")
    @Transactional(readOnly = true)
    public UserResponse getByUsername(String username) {
        return userRepository.findByUsername(username)
                .map(UserResponse::from)
                .orElseThrow(() -> new UsernameNotFoundException("User not found: " + username));
    }

    // Exercise 6: method-level rule, independent of the URL rule in SecurityConfig
    @PreAuthorize("hasRole('ADMIN')")
    @Transactional(readOnly = true)
    public List<UserResponse> getAll() {
        return userRepository.findAll().stream()
                .map(UserResponse::from)
                .toList();
    }

    // Exercise 4
    @Transactional
    public void changePassword(String username, String oldPassword, String newPassword) {
        User user = userRepository.findByUsername(username)
                .orElseThrow(() -> new UsernameNotFoundException("User not found: " + username));

        // The old password must match the stored BCrypt hash
        if (!passwordEncoder.matches(oldPassword, user.getPassword())) {
            throw new BusinessRuleException("Old password is incorrect");
        }
        if (passwordEncoder.matches(newPassword, user.getPassword())) {
            throw new BusinessRuleException("New password must be different from the old one");
        }

        // Store only the new HASH. The entity is managed, so the UPDATE runs on commit.
        user.setPassword(passwordEncoder.encode(newPassword));
    }
}
```

### `controller/AccountController.java`
```java
package com.example.jwtlesson.controller;

import com.example.jwtlesson.dto.ChangePasswordRequest;
import com.example.jwtlesson.service.UserService;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpStatus;
import org.springframework.security.core.Authentication;
import org.springframework.web.bind.annotation.PutMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.ResponseStatus;
import org.springframework.web.bind.annotation.RestController;

/** Under /api/user/** -> SecurityConfig requires ROLE_USER. */
@RestController
@RequestMapping("/api/user")
@RequiredArgsConstructor
public class AccountController {

    private final UserService userService;

    // PUT /api/user/password   {"oldPassword":"...","newPassword":"..."}   -> 204
    @PutMapping("/password")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void changePassword(Authentication authentication,
                               @Valid @RequestBody ChangePasswordRequest request) {
        // The username comes from the verified token, never from the request body
        userService.changePassword(authentication.getName(),
                request.oldPassword(), request.newPassword());
    }
}
```

**שימו לב:**
- הסיסמה הישנה שגויה מחזירה **400** ולא 401. 401 אומר "הטוקן לא תקין", והלקוח עלול לנתק את המשתמש בטעות.
- טוקנים שכבר הונפקו ממשיכים לעבוד אחרי החלפת סיסמה (מגבלה של JWT טהור). בתרגיל 7 אפשר לבטל את ה‑Refresh Tokens של המשתמש בעת החלפת סיסמה.

---

## תרגיל 5 (בינוני) – חסימת משתמש

הקוד נמצא ב‑`AdminUserService.setEnabled` וב‑`AdminController` (למעלה).

### `dto/SetEnabledRequest.java`
```java
package com.example.jwtlesson.dto;

import jakarta.validation.constraints.NotNull;

public record SetEnabledRequest(@NotNull Boolean enabled) {
}
```

### למה הטוקן מפסיק לעבוד מיד?
1. `JwtAuthFilter` טוען את המשתמש מה‑DB בכל בקשה.
2. `CustomUserDetailsService` בונה `UserDetails` עם `.disabled(!user.isEnabled())`.
3. ה‑Filter בודק `userDetails.isEnabled()`. אם false, הבקשה נשארת **אנונימית** ← `authenticationEntryPoint` ← **401**.
4. לוגין חדש נכשל כי `DaoAuthenticationProvider` זורק `DisabledException`, והיא יורשת מ‑`AuthenticationException`, ולכן מקבלים 401 עם אותה הודעה אחידה.

---

## תרגיל 6 (מתקדם) – `@EnableMethodSecurity` + `@PreAuthorize`

### שינוי ב‑`config/SecurityConfig.java`
מוסיפים אנוטציה אחת למחלקה, ו‑import אחד:
```java
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;

@Configuration
@EnableWebSecurity
@EnableMethodSecurity          // <-- NEW: turns on @PreAuthorize / @PostAuthorize / @Secured
@RequiredArgsConstructor
public class SecurityConfig {
    // ... unchanged
}
```

`@PreAuthorize` כבר הוצב ב‑`AdminUserService` (ברמת המחלקה) וב‑`UserService` (ברמת המתודה). ראו הקוד למעלה.

### ניסוי שמוכיח שזו שכבה נפרדת
1. ב‑`SecurityConfig` משנים זמנית את `.requestMatchers("/api/admin/**").hasRole("ADMIN")` ל‑`.requestMatchers("/api/admin/**").authenticated()`.
2. מריצים בקשה 9 (user ← `/api/admin/users`).
3. עדיין מקבלים **403**, כי `@PreAuthorize("hasRole('ADMIN')")` על `UserService.getAll()` חוסם גם כשחוק ה‑URL התרופף.
4. מחזירים את החוק.

### ההבדל מחוק ב‑`SecurityConfig`

| | חוק URL ב‑`SecurityConfig` | `@PreAuthorize` על מתודה |
|---|---|---|
| על מה חל | כתובת HTTP | מתודה, בכל מקום שקוראים לה |
| מתי נבדק | לפני ה‑Controller (ב‑Filter) | ברגע הקריאה למתודה (Proxy) |
| ביטוי | רק התאמת נתיב ותפקיד | SpEL מלא: פרמטרים, `authentication.name`, תנאים משולבים |
| מקרה טיפוסי | הגנה גסה לפי אזורים | הגנה עדינה, למשל "מנהל או הבעלים של המשאב" |
| חולשה | שוכחים לכסות כתובת חדשה, או שינוי נתיב מרפה את ההגנה | רק קריאות דרך ה‑Proxy מוגנות. `this.method()` פנימי עוקף את הבדיקה |

המלצה: להשתמש בשתיהן. חוק URL הוא חומת הגנה רחבה, ו‑`@PreAuthorize` מגן על הלוגיקה העסקית גם אם מישהו יקרא לה בעתיד מ‑Controller אחר, מ‑Scheduler או מ‑Message Listener.

> ה‑`AccessDeniedException` שנזרקת ממתודת Service מטופלת על ידי `ExceptionTranslationFilter` ומגיעה ל‑`accessDeniedHandler` שהגדרנו, כך שהתשובה היא אותו JSON של 403.

---

## תרגיל 7 (מתקדם) – Refresh Token

**עיצוב:**
- Access Token: JWT ל‑**15 דקות** (כמו קודם, רק קצר יותר).
- Refresh Token: מחרוזת אקראית **אטומה** (256 ביט) ל‑**7 ימים**, שמורה ב‑DB כ‑**Hash** של SHA‑256 (כמו סיסמה: מי שמקבל דליפת DB לא יכול להשתמש בטוקנים).
- **Rotation:** כל שימוש ב‑Refresh Token מבטל אותו ומנפיק זוג חדש.
- **זיהוי גניבה:** אם מישהו מציג Refresh Token שכבר בוטל, מבטלים את כל הטוקנים של המשתמש.
- **Logout אמיתי:** ביטול ה‑Refresh Token ב‑DB (ה‑Access Token הקצר פג בעצמו תוך 15 דקות).

### `application.properties` (שינוי)
```properties
# 15 minutes
jwt.expiration-ms=900000
# 7 days
jwt.refresh-expiration-ms=604800000
```

### `entity/RefreshToken.java`
```java
package com.example.jwtlesson.entity;

import jakarta.persistence.*;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

import java.time.Instant;

@Entity
@Table(name = "refresh_tokens")
@Getter
@Setter
@NoArgsConstructor
public class RefreshToken {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    // SHA-256 hex of the token (64 chars). The raw token is never stored.
    @Column(nullable = false, unique = true, length = 64)
    private String tokenHash;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "user_id")
    private User user;

    @Column(nullable = false)
    private Instant expiresAt;

    @Column(nullable = false)
    private boolean revoked;
}
```

### `repository/RefreshTokenRepository.java`
```java
package com.example.jwtlesson.repository;

import com.example.jwtlesson.entity.RefreshToken;
import com.example.jwtlesson.entity.User;
import org.springframework.data.jpa.repository.EntityGraph;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import java.util.Optional;

public interface RefreshTokenRepository extends JpaRepository<RefreshToken, Long> {

    @EntityGraph(attributePaths = "user")
    Optional<RefreshToken> findByTokenHash(String tokenHash);

    @Modifying
    @Query("update RefreshToken t set t.revoked = true where t.user = :user")
    int revokeAllForUser(@Param("user") User user);
}
```

### `service/RefreshTokenService.java`
```java
package com.example.jwtlesson.service;

import com.example.jwtlesson.entity.RefreshToken;
import com.example.jwtlesson.entity.User;
import com.example.jwtlesson.exception.InvalidRefreshTokenException;
import com.example.jwtlesson.repository.RefreshTokenRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.security.SecureRandom;
import java.time.Instant;
import java.util.Base64;
import java.util.HexFormat;

@Service
@RequiredArgsConstructor
public class RefreshTokenService {

    private static final SecureRandom RANDOM = new SecureRandom();

    private final RefreshTokenRepository repository;

    @Value("${jwt.refresh-expiration-ms}")
    private long refreshExpirationMs;

    /** Creates a new refresh token and returns the RAW value (the only time it is visible). */
    @Transactional
    public String create(User user) {
        byte[] bytes = new byte[32];                 // 256 random bits
        RANDOM.nextBytes(bytes);
        String raw = Base64.getUrlEncoder().withoutPadding().encodeToString(bytes);

        RefreshToken token = new RefreshToken();
        token.setTokenHash(hash(raw));
        token.setUser(user);
        token.setExpiresAt(Instant.now().plusMillis(refreshExpirationMs));
        token.setRevoked(false);
        repository.save(token);

        return raw;
    }

    /**
     * Validates the token and revokes it (rotation: every refresh token works once).
     * noRollbackFor: the "revoke everything" in the reuse case must be committed
     * even though we throw an exception right after it.
     */
    @Transactional(noRollbackFor = InvalidRefreshTokenException.class)
    public User consume(String rawToken) {
        RefreshToken token = repository.findByTokenHash(hash(rawToken))
                .orElseThrow(InvalidRefreshTokenException::new);

        if (token.isRevoked()) {
            // An already-used token came back -> it may have been stolen. Kill all sessions of this user.
            repository.revokeAllForUser(token.getUser());
            throw new InvalidRefreshTokenException();
        }
        if (token.getExpiresAt().isBefore(Instant.now()) || !token.getUser().isEnabled()) {
            throw new InvalidRefreshTokenException();
        }

        token.setRevoked(true);
        return token.getUser();
    }

    /** Logout. Silent if the token is unknown (no information leak). */
    @Transactional
    public void revoke(String rawToken) {
        repository.findByTokenHash(hash(rawToken)).ifPresent(t -> t.setRevoked(true));
    }

    private static String hash(String raw) {
        try {
            byte[] digest = MessageDigest.getInstance("SHA-256")
                    .digest(raw.getBytes(StandardCharsets.UTF_8));
            return HexFormat.of().formatHex(digest);
        } catch (NoSuchAlgorithmException e) {
            throw new IllegalStateException(e);   // SHA-256 exists in every JVM
        }
    }
}
```

### `dto/RefreshRequest.java`
```java
package com.example.jwtlesson.dto;

import jakarta.validation.constraints.NotBlank;

public record RefreshRequest(@NotBlank String refreshToken) {
}
```

### `dto/LoginResponse.java` (מחליף את הקיים)
```java
package com.example.jwtlesson.dto;

import java.util.List;

public record LoginResponse(
        String token,            // access token (name kept so the existing .http tests still pass)
        String refreshToken,
        String tokenType,        // "Bearer"
        long expiresInMs,        // lifetime of the ACCESS token
        String username,
        List<String> roles) {
}
```

### `service/UserLoginService.java` (מחליף את הקיים)
```java
package com.example.jwtlesson.service;

import com.example.jwtlesson.dto.LoginRequest;
import com.example.jwtlesson.dto.LoginResponse;
import com.example.jwtlesson.dto.RefreshRequest;
import com.example.jwtlesson.dto.RegisterRequest;
import com.example.jwtlesson.dto.UserResponse;
import com.example.jwtlesson.entity.Role;
import com.example.jwtlesson.entity.RoleName;
import com.example.jwtlesson.entity.User;
import com.example.jwtlesson.exception.UsernameTakenException;
import com.example.jwtlesson.repository.RoleRepository;
import com.example.jwtlesson.repository.UserRepository;
import com.example.jwtlesson.security.CustomUserDetailsService;
import com.example.jwtlesson.security.JwtUtil;
import lombok.RequiredArgsConstructor;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.HashSet;
import java.util.List;
import java.util.Set;

@Service
@RequiredArgsConstructor
public class UserLoginService {

    private final AuthenticationManager authenticationManager;
    private final JwtUtil jwtUtil;
    private final UserRepository userRepository;
    private final RoleRepository roleRepository;
    private final PasswordEncoder passwordEncoder;
    private final RefreshTokenService refreshTokenService;
    private final CustomUserDetailsService userDetailsService;

    /** Name + password -> access token (15 min) + refresh token (7 days). */
    public LoginResponse login(LoginRequest request) {
        Authentication authentication = authenticationManager.authenticate(
                new UsernamePasswordAuthenticationToken(request.username(), request.password()));

        UserDetails userDetails = (UserDetails) authentication.getPrincipal();
        User user = userRepository.findByUsername(userDetails.getUsername())
                .orElseThrow(() -> new IllegalStateException("Authenticated user missing in DB"));

        return buildResponse(userDetails, refreshTokenService.create(user));
    }

    /** Refresh token -> NEW access token + NEW refresh token (rotation). */
    public LoginResponse refresh(RefreshRequest request) {
        User user = refreshTokenService.consume(request.refreshToken());

        // Reload roles from the DB so a role change is reflected in the new token
        UserDetails userDetails = userDetailsService.loadUserByUsername(user.getUsername());

        return buildResponse(userDetails, refreshTokenService.create(user));
    }

    /** Real logout: the refresh token can never be used again. */
    public void logout(RefreshRequest request) {
        refreshTokenService.revoke(request.refreshToken());
    }

    /** New users always get ROLE_USER only. Admins are never created through this endpoint. */
    @Transactional
    public UserResponse register(RegisterRequest request) {
        if (userRepository.existsByUsername(request.username())) {
            throw new UsernameTakenException(request.username());
        }

        Role userRole = roleRepository.findByName(RoleName.ROLE_USER)
                .orElseThrow(() -> new IllegalStateException("ROLE_USER missing - did DataInitializer run?"));

        User user = User.builder()
                .username(request.username())
                .password(passwordEncoder.encode(request.password()))
                .roles(new HashSet<>(Set.of(userRole)))
                .build();

        return UserResponse.from(userRepository.save(user));
    }

    private LoginResponse buildResponse(UserDetails userDetails, String refreshToken) {
        List<String> roles = userDetails.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority)
                .toList();

        return new LoginResponse(
                jwtUtil.generateToken(userDetails),
                refreshToken,
                "Bearer",
                jwtUtil.getExpirationMs(),
                userDetails.getUsername(),
                roles);
    }
}
```

### `controller/UserLoginController.java` (מחליף את הקיים)
```java
package com.example.jwtlesson.controller;

import com.example.jwtlesson.dto.LoginRequest;
import com.example.jwtlesson.dto.LoginResponse;
import com.example.jwtlesson.dto.RefreshRequest;
import com.example.jwtlesson.dto.RegisterRequest;
import com.example.jwtlesson.dto.UserResponse;
import com.example.jwtlesson.service.UserLoginService;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.*;

/** Public endpoints (permitAll for /api/auth/** in SecurityConfig). */
@RestController
@RequestMapping("/api/auth")
@RequiredArgsConstructor
public class UserLoginController {

    private final UserLoginService userLoginService;

    @PostMapping("/login")
    public LoginResponse login(@Valid @RequestBody LoginRequest request) {
        return userLoginService.login(request);
    }

    @PostMapping("/register")
    @ResponseStatus(HttpStatus.CREATED)
    public UserResponse register(@Valid @RequestBody RegisterRequest request) {
        return userLoginService.register(request);
    }

    // POST /api/auth/refresh   {"refreshToken":"..."}   -> new access + new refresh token
    @PostMapping("/refresh")
    public LoginResponse refresh(@Valid @RequestBody RefreshRequest request) {
        return userLoginService.refresh(request);
    }

    // POST /api/auth/logout   {"refreshToken":"..."}   -> 204
    @PostMapping("/logout")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void logout(@Valid @RequestBody RefreshRequest request) {
        userLoginService.logout(request);
    }
}
```

**אין צורך לשנות** את `JwtAuthFilter` או את `SecurityConfig`:
- `/api/auth/**` כבר `permitAll`, אז `/refresh` ו‑`/logout` פתוחים.
- Refresh Token אינו JWT, ולכן אם מישהו ישלח אותו בכותרת `Authorization: Bearer` ה‑Filter ידחה אותו (`MalformedJwtException`) והבקשה תישאר אנונימית.

> הערת ייצור: כדאי להוסיף משימה מתוזמנת (`@Scheduled`) שמוחקת Refresh Tokens שפג תוקפם, ולבטל את כל ה‑Refresh Tokens של משתמש גם בהחלפת סיסמה (`revokeAllForUser`).

---

## קובץ הבדיקות – `http/exercises-tests.http`

מריצים **אחרי** `security-tests.http` (או לבד: הוא מבצע לוגין בעצמו) עם הסביבה `dev`, `Run All Requests in File`.

```http
###############################################################################
# Exercises 1, 3, 4, 5, 7 - REST tests for the IntelliJ IDEA HTTP Client
# Run top to bottom: later requests use values saved by earlier ones.
###############################################################################


### SETUP A. Login as admin
POST {{host}}/api/auth/login
Content-Type: application/json

{ "username": "admin", "password": "Admin123!" }

> {%
    client.test("A. admin login", function () {
        client.assert(response.status === 200, "Expected 200, got " + response.status);
    });
    client.global.set("adminToken", response.body.token);
%}


### SETUP B. Login as user
POST {{host}}/api/auth/login
Content-Type: application/json

{ "username": "user", "password": "User123!" }

> {%
    client.test("B. user login", function () {
        client.assert(response.status === 200, "Expected 200, got " + response.status);
    });
    client.global.set("userToken", response.body.token);
%}


###############################################################################
# EXERCISE 1 - GET /api/admin/stats
###############################################################################

### 1.1 USER on the stats endpoint -> 403
GET {{host}}/api/admin/stats
Authorization: Bearer {{userToken}}

> {%
    client.test("1.1 user -> 403", function () {
        client.assert(response.status === 403, "Expected 403, got " + response.status);
    });
%}


### 1.2 ADMIN on the stats endpoint -> 200 + number of users
GET {{host}}/api/admin/stats
Authorization: Bearer {{adminToken}}

> {%
    client.test("1.2 admin -> 200 with totalUsers", function () {
        client.assert(response.status === 200, "Expected 200, got " + response.status);
        client.assert(typeof response.body.totalUsers === "number", "totalUsers must be a number");
        client.assert(response.body.totalUsers >= 2, "At least admin and user exist");
    });
%}


###############################################################################
# EXERCISE 3 - admin adds a role, effective WITHOUT logging in again
###############################################################################

### 3.0 Register a victim user
POST {{host}}/api/auth/register
Content-Type: application/json

{ "username": "victim_{{$timestamp}}", "password": "Victim123!" }

> {%
    client.test("3.0 register -> 201", function () {
        client.assert(response.status === 201, "Expected 201, got " + response.status);
    });
    client.global.set("victimId", response.body.id);
    client.global.set("victimName", response.body.username);
%}


### 3.1 Victim logs in (the token is saved)
POST {{host}}/api/auth/login
Content-Type: application/json

{ "username": "{{victimName}}", "password": "Victim123!" }

> {%
    client.test("3.1 victim login -> 200", function () {
        client.assert(response.status === 200, "Expected 200, got " + response.status);
    });
    client.global.set("victimToken", response.body.token);
%}


### 3.2 Victim is NOT an admin yet -> 403
GET {{host}}/api/admin/users
Authorization: Bearer {{victimToken}}

> {%
    client.test("3.2 victim on admin endpoint -> 403", function () {
        client.assert(response.status === 403, "Expected 403, got " + response.status);
    });
%}


### 3.3 Admin gives the victim ROLE_ADMIN -> 200, two roles
PUT {{host}}/api/admin/users/{{victimId}}/roles
Authorization: Bearer {{adminToken}}
Content-Type: application/json

{ "role": "ROLE_ADMIN" }

> {%
    client.test("3.3 role added", function () {
        client.assert(response.status === 200, "Expected 200, got " + response.status);
        client.assert(response.body.roles.indexOf("ROLE_ADMIN") >= 0, "ROLE_ADMIN expected");
        client.assert(response.body.roles.indexOf("ROLE_USER") >= 0, "ROLE_USER must remain");
        client.assert(response.body.password === undefined, "Password must never be returned");
    });
%}


### 3.4 SAME OLD TOKEN, no new login -> 200 (roles are loaded from the DB on every request)
GET {{host}}/api/admin/users
Authorization: Bearer {{victimToken}}

> {%
    client.test("3.4 old token works with the new role", function () {
        client.assert(response.status === 200, "Expected 200, got " + response.status);
    });
%}


### 3.5 A regular user tries to grant roles -> 403
PUT {{host}}/api/admin/users/{{victimId}}/roles
Authorization: Bearer {{userToken}}
Content-Type: application/json

{ "role": "ROLE_ADMIN" }

> {%
    client.test("3.5 user cannot grant roles -> 403", function () {
        client.assert(response.status === 403, "Expected 403, got " + response.status);
    });
%}


### 3.6 Invalid role name -> 400
PUT {{host}}/api/admin/users/{{victimId}}/roles
Authorization: Bearer {{adminToken}}
Content-Type: application/json

{ "role": "ROLE_SUPERMAN" }

> {%
    client.test("3.6 invalid role -> 400", function () {
        client.assert(response.status === 400, "Expected 400, got " + response.status);
    });
%}


### 3.7 Unknown user id -> 404
PUT {{host}}/api/admin/users/999999/roles
Authorization: Bearer {{adminToken}}
Content-Type: application/json

{ "role": "ROLE_USER" }

> {%
    client.test("3.7 unknown id -> 404", function () {
        client.assert(response.status === 404, "Expected 404, got " + response.status);
    });
%}


###############################################################################
# EXERCISE 4 - PUT /api/user/password
###############################################################################

### 4.1 Wrong OLD password -> 400
PUT {{host}}/api/user/password
Authorization: Bearer {{victimToken}}
Content-Type: application/json

{ "oldPassword": "not-the-password", "newPassword": "NewPass123!" }

> {%
    client.test("4.1 wrong old password -> 400", function () {
        client.assert(response.status === 400, "Expected 400, got " + response.status);
        client.assert(response.body.error === "Old password is incorrect", "Unexpected message");
    });
%}


### 4.2 New password too short -> 400
PUT {{host}}/api/user/password
Authorization: Bearer {{victimToken}}
Content-Type: application/json

{ "oldPassword": "Victim123!", "newPassword": "123" }

> {%
    client.test("4.2 short new password -> 400", function () {
        client.assert(response.status === 400, "Expected 400, got " + response.status);
        client.assert(response.body.fields.newPassword !== undefined, "Expected an error on 'newPassword'");
    });
%}


### 4.3 New password equal to the old one -> 400
PUT {{host}}/api/user/password
Authorization: Bearer {{victimToken}}
Content-Type: application/json

{ "oldPassword": "Victim123!", "newPassword": "Victim123!" }

> {%
    client.test("4.3 same password -> 400", function () {
        client.assert(response.status === 400, "Expected 400, got " + response.status);
    });
%}


### 4.4 Successful change -> 204
PUT {{host}}/api/user/password
Authorization: Bearer {{victimToken}}
Content-Type: application/json

{ "oldPassword": "Victim123!", "newPassword": "NewPass123!" }

> {%
    client.test("4.4 password changed -> 204", function () {
        client.assert(response.status === 204, "Expected 204, got " + response.status);
    });
%}


### 4.5 Login with the OLD password -> 401
POST {{host}}/api/auth/login
Content-Type: application/json

{ "username": "{{victimName}}", "password": "Victim123!" }

> {%
    client.test("4.5 old password no longer works -> 401", function () {
        client.assert(response.status === 401, "Expected 401, got " + response.status);
    });
%}


### 4.6 Login with the NEW password -> 200 (token saved)
POST {{host}}/api/auth/login
Content-Type: application/json

{ "username": "{{victimName}}", "password": "NewPass123!" }

> {%
    client.test("4.6 new password works -> 200", function () {
        client.assert(response.status === 200, "Expected 200, got " + response.status);
    });
    client.global.set("victimToken", response.body.token);
%}


### 4.7 Change password without a token -> 401
PUT {{host}}/api/user/password
Content-Type: application/json

{ "oldPassword": "NewPass123!", "newPassword": "Another123!" }

> {%
    client.test("4.7 no token -> 401", function () {
        client.assert(response.status === 401, "Expected 401, got " + response.status);
    });
%}


###############################################################################
# EXERCISE 5 - block a user, the token stops working immediately
###############################################################################

### 5.1 Victim's token works before the block -> 200
GET {{host}}/api/user/me
Authorization: Bearer {{victimToken}}

> {%
    client.test("5.1 token works before the block", function () {
        client.assert(response.status === 200, "Expected 200, got " + response.status);
    });
%}


### 5.2 Admin disables the victim -> 200, enabled=false
PUT {{host}}/api/admin/users/{{victimId}}/enabled
Authorization: Bearer {{adminToken}}
Content-Type: application/json

{ "enabled": false }

> {%
    client.test("5.2 user disabled", function () {
        client.assert(response.status === 200, "Expected 200, got " + response.status);
        client.assert(response.body.enabled === false, "enabled must be false");
    });
%}


### 5.3 THE SAME token stops working immediately -> 401
GET {{host}}/api/user/me
Authorization: Bearer {{victimToken}}

> {%
    client.test("5.3 blocked user's token -> 401", function () {
        client.assert(response.status === 401, "Expected 401, got " + response.status);
    });
%}


### 5.4 Blocked user cannot log in -> 401, same generic message
POST {{host}}/api/auth/login
Content-Type: application/json

{ "username": "{{victimName}}", "password": "NewPass123!" }

> {%
    client.test("5.4 blocked login -> 401, no information leak", function () {
        client.assert(response.status === 401, "Expected 401, got " + response.status);
        client.assert(response.body.error === "Invalid username or password", "Unexpected message");
    });
%}


### 5.5 Admin enables the victim again -> 200
PUT {{host}}/api/admin/users/{{victimId}}/enabled
Authorization: Bearer {{adminToken}}
Content-Type: application/json

{ "enabled": true }

> {%
    client.test("5.5 user enabled again", function () {
        client.assert(response.status === 200, "Expected 200, got " + response.status);
        client.assert(response.body.enabled === true, "enabled must be true");
    });
%}


### 5.6 The old token works again (the token never changed, only the DB did) -> 200
GET {{host}}/api/user/me
Authorization: Bearer {{victimToken}}

> {%
    client.test("5.6 token works again", function () {
        client.assert(response.status === 200, "Expected 200, got " + response.status);
    });
%}


### 5.7a Get the admin's own id
GET {{host}}/api/user/me
Authorization: Bearer {{adminToken}}

> {%
    client.global.set("adminId", response.body.id);
%}


### 5.7b Admin tries to disable himself -> 400
PUT {{host}}/api/admin/users/{{adminId}}/enabled
Authorization: Bearer {{adminToken}}
Content-Type: application/json

{ "enabled": false }

> {%
    client.test("5.7 admin cannot disable himself -> 400", function () {
        client.assert(response.status === 400, "Expected 400, got " + response.status);
    });
%}


### 5.8 Regular user tries to disable someone -> 403
PUT {{host}}/api/admin/users/{{victimId}}/enabled
Authorization: Bearer {{userToken}}
Content-Type: application/json

{ "enabled": false }

> {%
    client.test("5.8 user cannot disable -> 403", function () {
        client.assert(response.status === 403, "Expected 403, got " + response.status);
    });
%}


###############################################################################
# EXERCISE 7 - Refresh Token (access = 15 min, refresh = 7 days)
###############################################################################

### 7.1 Login returns BOTH tokens
POST {{host}}/api/auth/login
Content-Type: application/json

{ "username": "user", "password": "User123!" }

> {%
    client.test("7.1 login returns access + refresh token", function () {
        client.assert(response.status === 200, "Expected 200, got " + response.status);
        client.assert(response.body.token.split(".").length === 3, "access token is a JWT");
        client.assert(typeof response.body.refreshToken === "string" && response.body.refreshToken.length > 20, "refreshToken expected");
        client.assert(response.body.expiresInMs === 900000, "access token must live 15 minutes");
    });
    client.global.set("rt1", response.body.refreshToken);
%}


### 7.2 Refresh -> new access token + NEW refresh token (rotation)
POST {{host}}/api/auth/refresh
Content-Type: application/json

{ "refreshToken": "{{rt1}}" }

> {%
    client.test("7.2 refresh -> 200 with a rotated refresh token", function () {
        client.assert(response.status === 200, "Expected 200, got " + response.status);
        client.assert(response.body.refreshToken !== client.global.get("rt1"), "refresh token must be rotated");
        client.assert(response.body.username === "user", "Expected username 'user'");
    });
    client.global.set("rt2", response.body.refreshToken);
    client.global.set("refreshedAccessToken", response.body.token);
%}


### 7.3 The NEW access token works -> 200
GET {{host}}/api/user/me
Authorization: Bearer {{refreshedAccessToken}}

> {%
    client.test("7.3 refreshed access token works", function () {
        client.assert(response.status === 200, "Expected 200, got " + response.status);
    });
%}


### 7.4 Re-using the OLD refresh token -> 401 (and it revokes ALL tokens of this user)
POST {{host}}/api/auth/refresh
Content-Type: application/json

{ "refreshToken": "{{rt1}}" }

> {%
    client.test("7.4 reused refresh token -> 401", function () {
        client.assert(response.status === 401, "Expected 401, got " + response.status);
    });
%}


### 7.5 Because of the reuse, even the newest refresh token is revoked now -> 401
POST {{host}}/api/auth/refresh
Content-Type: application/json

{ "refreshToken": "{{rt2}}" }

> {%
    client.test("7.5 theft detection revoked the whole family -> 401", function () {
        client.assert(response.status === 401, "Expected 401, got " + response.status);
    });
%}


### 7.6 Login again, then logout
POST {{host}}/api/auth/login
Content-Type: application/json

{ "username": "user", "password": "User123!" }

> {%
    client.global.set("rt3", response.body.refreshToken);
%}


### 7.7 Logout -> 204
POST {{host}}/api/auth/logout
Content-Type: application/json

{ "refreshToken": "{{rt3}}" }

> {%
    client.test("7.7 logout -> 204", function () {
        client.assert(response.status === 204, "Expected 204, got " + response.status);
    });
%}


### 7.8 Refresh after logout -> 401
POST {{host}}/api/auth/refresh
Content-Type: application/json

{ "refreshToken": "{{rt3}}" }

> {%
    client.test("7.8 refresh after logout -> 401", function () {
        client.assert(response.status === 401, "Expected 401, got " + response.status);
    });
%}


### 7.9 Garbage refresh token -> 401
POST {{host}}/api/auth/refresh
Content-Type: application/json

{ "refreshToken": "this-is-not-a-real-token" }

> {%
    client.test("7.9 garbage refresh token -> 401", function () {
        client.assert(response.status === 401, "Expected 401, got " + response.status);
    });
%}


### 7.10 Empty body -> 400
POST {{host}}/api/auth/refresh
Content-Type: application/json

{ "refreshToken": "" }

> {%
    client.test("7.10 empty refresh token -> 400", function () {
        client.assert(response.status === 400, "Expected 400, got " + response.status);
    });
%}


### 7.11 A refresh token must NOT be accepted as an access token -> 401
GET {{host}}/api/user/me
Authorization: Bearer {{rt3}}

> {%
    client.test("7.11 refresh token as access token -> 401", function () {
        client.assert(response.status === 401, "Expected 401, got " + response.status);
    });
%}
```

---

## בדיקת DB לתרגיל 7
```sql
SELECT u.username, left(rt.token_hash, 10) AS hash_prefix, rt.revoked, rt.expires_at
FROM refresh_tokens rt
JOIN users u ON u.id = rt.user_id
ORDER BY rt.id DESC;
```
בעמודה `token_hash` רואים Hash בלבד, אף פעם לא את הטוקן עצמו.

## סדר עבודה מומלץ
1. תרגילים 1, 3, 4, 5, 6: מוסיפים את הקבצים החדשים ומחליפים את `GlobalExceptionHandler`, `UserService`. מוסיפים `@EnableMethodSecurity`.
2. מריצים `exercises-tests.http` עד בדיקה 5.8. כולן צריכות להיות ירוקות.
3. תרגיל 7: מוסיפים Entity, Repository, Services, DTOs ומחליפים `LoginResponse`, `UserLoginService`, `UserLoginController` ואת ה‑properties. אחר כך מריצים את בדיקות 7.x ואת `security-tests.http` המקורי כדי לוודא שלא נשבר דבר (השדה `token` נשמר).
4. תרגיל 2: `mvn test` להרצת `JwtUtilExpiryTest`.
