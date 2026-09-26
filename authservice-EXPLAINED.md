# authservice — EXPLAINED

## What it does (the whole point)

`authservice` owns **identity**: sign up, log in, and hand out **JWT access tokens**
+ **refresh tokens**. Other services never see passwords — they just trust a valid JWT.
When a new user signs up, authservice also **publishes a Kafka event** so `userservice`
can create the matching profile. It's a Spring Boot app on port **9898** with a MySQL
database and a Kafka **producer**.

This service is already implemented — this doc is your reference while you build `userservice`.

---

## Jargon, explained once

| Term | Plain meaning |
|---|---|
| **JWT** (JSON Web Token) | A signed string the client sends on every request (`Authorization: Bearer <token>`). The server verifies the signature with a secret key — no DB lookup needed. Contains the username ("subject") and an expiry. |
| **access token** | Short-lived JWT (~100 min here). Proves who you are. |
| **refresh token** | Long-lived random UUID stored in the DB. When the access token expires, you exchange the refresh token for a fresh access token without logging in again. |
| **Spring Security filter chain** | A pipeline every HTTP request passes through before hitting your controller. Each filter can inspect/reject/authenticate the request. |
| **`OncePerRequestFilter`** | Base class for a custom filter that runs exactly once per request. `JwtAuthFilter` is one. |
| **`UserDetailsService`** | Spring Security's "load a user by username" interface. Returns a `UserDetails` (username + hashed password + roles). |
| **`AuthenticationManager` / `AuthenticationProvider`** | The thing that actually checks "does this username+password match?". `DaoAuthenticationProvider` does it against a `UserDetailsService` + `PasswordEncoder`. |
| **`BCryptPasswordEncoder`** | Hashes passwords one-way. You never store or compare plaintext. |
| **`SecurityContextHolder`** | A per-request holder for "who is the current authenticated user". A filter sets it; controllers read it. |
| **`SessionCreationPolicy.STATELESS`** | No server-side session/cookie. Every request must carry its own JWT. |
| **`@Entity` / JPA / Hibernate** | Java class ↔ DB table. Hibernate generates the SQL. |
| **`CrudRepository`** | Spring Data interface giving `save`, `findById`, `delete`, … for free. Add `findByX` methods and Spring writes the query from the name. |
| **DTO** | Plain class for data crossing a boundary (HTTP body, Kafka message), separate from the DB entity. |
| **Kafka producer / serializer** | Producer writes messages to a topic. Serializer turns the Java object into bytes first (`UserInfoSerializer` here — JSON via Jackson). |
| **Lombok** (`@Data`, `@Builder`, `@AllArgsConstructor`…) | Generates getters/setters/constructors/builders at compile time. |
| **`@Value("${...}")`** | Injects a value from `application.properties` into a field. |

---

## Request flow — the three main routes

### 1. `POST /auth/v1/signup`  → new account
```
TokenController? no — AuthController.SignUp(UserInfoDto)   (controller/AuthController.java)
 └─ UserDetailsServiceImpl.signupUser(dto)                 (service/UserDetailsServiceImpl.java)
      ├─ passwordEncoder.encode(password)          (BCrypt hash)
      ├─ check UserRepository.findByUsername(...)   → already exists? return null → 400 "Already Exist"
      ├─ UserRepository.save(new UserInfo(uuid, username, hash, roles))   (INSERT into `users`)
      └─ userInfoProducer.sendEventToKafka(UserInfoEvent)   → Kafka topic "user_service"
 └─ RefreshTokenService.createRefreshToken(username)  → row in `tokens` table
 └─ JwtService.GenerateToken(username)                → signed JWT
 └─ 200 { accessToken, token (=refresh), userId }
```

### 2. `POST /auth/v1/login`  → exchange username+password for tokens
```
TokenController.AuthenticateAndGetToken(AuthRequestDTO)    (controller/TokenController.java)
 └─ authenticationManager.authenticate(username, password)
      └─ DaoAuthenticationProvider
           └─ UserDetailsServiceImpl.loadUserByUsername(username)  → CustomUserDetails
           └─ BCrypt compares submitted password vs stored hash
 └─ authenticated? RefreshTokenService.createRefreshToken(...) + JwtService.GenerateToken(...)
 └─ 200 { accessToken, token }
```

### 3. Any protected request, e.g. `GET /auth/v1/ping`
```
Request with header:  Authorization: Bearer <jwt>
 └─ JwtAuthFilter.doFilterInternal(...)          (auth/JwtAuthFilter.java)  — runs before the controller
      ├─ pull token out of the header
      ├─ JwtService.extractUsername(token)        (verifies signature + reads "subject")
      ├─ UserDetailsServiceImpl.loadUserByUsername(username)   (DB lookup)
      ├─ JwtService.validateToken(token, userDetails)   (username matches + not expired)
      └─ set SecurityContextHolder authentication
 └─ SecurityConfig filter chain: .anyRequest().authenticated() → allowed because context is set
 └─ AuthController.ping() reads SecurityContextHolder → returns the userId
```

### 4. `POST /auth/v1/refreshToken`
```
TokenController.refreshToken(RefreshTokenRequestDTO)
 └─ RefreshTokenService.findByToken(token)      (SELECT from `tokens`)
 └─ .verifyExpiration(...)   expired? delete row + throw. ok? continue.
 └─ JwtService.GenerateToken(username)          → new access token
 └─ { accessToken, token (same refresh) }
```

---

## Every file, one line

| File | Role |
|---|---|
| `App.java` | Entry point. `@SpringBootApplication` + `@EnableJpaRepositories`. |
| `controller/AuthController.java` | `POST /auth/v1/signup`, `GET /auth/v1/ping`, `GET /health`. |
| `controller/TokenController.java` | `POST /auth/v1/login`, `POST /auth/v1/refreshToken`. |
| `controller/SecurityConfig.java` | The Spring Security setup: which URLs are public, stateless sessions, register `JwtAuthFilter`, wire the `AuthenticationProvider`. |
| `auth/JwtAuthFilter.java` | Runs on every request; turns a valid `Bearer` token into an authenticated user. |
| `auth/UserConfig.java` | One `@Bean`: the `BCryptPasswordEncoder`. |
| `service/JwtService.java` | Create / parse / validate JWTs. Uses `jwt.secret`. |
| `service/RefreshTokenService.java` | Create refresh tokens, look them up, expire them. Table `tokens`. |
| `service/UserDetailsServiceImpl.java` | Load user for Spring Security **and** the signup logic (hash password, save, publish Kafka event). |
| `service/CustomUserDetails.java` | Adapts a `UserInfo` row into the `UserDetails` shape Spring Security wants (with roles as authorities). |
| `entities/UserInfo.java` | `@Entity` — `users` table. `userId` (UUID string) PK, username, password hash, roles (`@ManyToMany`). |
| `entities/UserRole.java` | `@Entity` — `roles` table. |
| `entities/RefreshToken.java` | `@Entity` — `tokens` table. Token UUID + expiry + `@OneToOne` to the user. |
| `repository/UserRepository.java` | `CrudRepository<UserInfo,String>` + `findByUsername`. |
| `repository/RefreshTokenRepository.java` | `CrudRepository<RefreshToken,Integer>` + `findByToken`. |
| `model/UserInfoDto.java` | Signup request body (extends `UserInfo`, adds firstName/lastName/email/phone). |
| `request/AuthRequestDTO.java` | Login request body (username, password). |
| `request/RefreshTokenRequestDTO.java` | Refresh request body (token). |
| `response/JwtResponseDTO.java` | Response body: `accessToken`, `token`, `userId`. |
| `eventProducer/UserInfoEvent.java` | The Kafka message shape (firstName, lastName, email, phoneNumber, userId). |
| `eventProducer/UserInfoProducer.java` | Sends a `UserInfoEvent` to the topic via `KafkaTemplate`. |
| `serializer/UserInfoSerializer.java` | `UserInfoEvent` → JSON bytes for Kafka (Jackson `ObjectMapper`). |
| `utils/ValidationUtil.java` | Empty stub — validation was planned but not implemented. |
| `app/src/main/resources/application.properties` | Config (below). **Git-ignored** — it holds real credentials; `application.properties.example` is the committed template. |
| `Dockerfile` | Packages `app/build/libs/app.jar` onto `amazoncorretto:21`, exposes 9898. |
| `services.yml` | docker-compose for the whole stack: zookeeper, kafka, mysql, userservice, authservice, kong gateway. |
| `cloudformation-template.yaml` | AWS deploy: autoscaling group + load balancer, container gets DB/Kafka hosts as env vars. |
| `.github/workflows/deploy.yml` | On push to `main`: build image → push to ECR → deploy the CloudFormation stack. Secrets come from GitHub Actions. |

---

## Every config value (`app/src/main/resources/application.properties`)

| Line | Controls | Note |
|---|---|---|
| `spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver` | JDBC driver | MySQL |
| `spring.datasource.url=jdbc:mysql://${MYSQL_HOST:192.168.122.132}:${MYSQL_PORT:3306}/${MYSQL_DB:authservice}?...` | DB host/port/schema | **`192.168.122.132` is YOUR VM** (see below). `createDatabaseIfNotExist=true` auto-creates the schema. |
| `spring.datasource.username=${MYSQL_USER:varun}` | DB user | **Yours** |
| `spring.datasource.password=${MYSQL_PASSWORD:varunkewlani}` | DB password | **Yours**, plaintext in the file — that's why it's git-ignored |
| `spring.jpa.show-sql=true` | Log SQL | Debugging |
| `#spring.jpa.hibernate.ddl-auto=update` (commented) | — | Comment says: uncomment after tables exist so they aren't dropped on restart |
| `spring.jpa.properties.hibernate.dialect=...MySQL8Dialect` | SQL flavour Hibernate emits | MySQL 8 |
| `spring.jpa.properties.hibernate.hbm2ddl.auto=update` | Create/alter tables to match entities, **keep data** | Safe-ish for dev |
| `server.port=9898` | HTTP port | — |
| `logging.level.*=DEBUG/TRACE` | Verbose logs (security, web, SQL, Hikari, kafka) | Learning; noisy |
| `spring.datasource.hikari.maximum-pool-size=20` / `minimum-idle=10` | DB connection pool | small-app defaults |
| `spring.kafka.producer.bootstrap-servers=192.168.122.132:${KAFKA_PORT:9092}` | Kafka broker address | **Host hardcoded to YOUR VM** — not even using an env var |
| `spring.kafka.producer.properties.max.in.flight.requests.per.connection=1` | Only 1 unacked request at a time | Preserves message order on retry |
| `spring.kafka.producer.properties.retries=3` | Retry a failed send 3× | — |
| `spring.kafka.producer.properties.acks=all` | Wait for all in-sync replicas to confirm | Most durable setting |
| `spring.kafka.producer.key-serializer=...StringSerializer` | Message key → bytes | — |
| `spring.kafka.producer.value-serializer=authservice.serializer.UserInfoSerializer` | Message value → bytes | The custom JSON serializer |
| `spring.kafka.topic-json.name=user_service` | Topic to publish to | **Must match userservice's** `spring.kafka.topic-json.name` |
| `spring.kafka.producer.properties.spring.json.type.mapping=auth:authservice.model.UserInfoEvent` | Short type name → class in the header | Note: class is actually in `authservice.eventProducer`, not `.model` — but the custom serializer ignores this, so it doesn't bite |
| `security.basic.enable=false` / `security.ignored=/**` | Legacy Spring Boot 1.x security props | No effect on this Spring Boot 3 app — leftover |
| `jwt.secret=${JWT_SECRET:3576...4629}` | HMAC key that signs/verifies every JWT | The `357638...` default is a **well-known tutorial sample** — fine for learning, **must be replaced** for anything real (`openssl rand -hex 32`). Same secret must be used everywhere that verifies these tokens. |

---

## Things that talk to other systems — and what breaks if wrong

| Talks to | Via | If misconfigured |
|---|---|---|
| **MySQL** (`authservice` schema) | `spring.datasource.*` | App won't start; every signup/login fails. |
| **Kafka** | producer config, topic `user_service` | App still starts and login/signup still return tokens, but `userservice` **never hears about new users** → no profile rows. Failures show in logs as producer timeouts. |
| **userservice** | *Only indirectly, through Kafka.* No HTTP call. | Topic name / broker mismatch between the two = they never connect. |
| **AWS (ECR, CloudFormation)** | `.github/workflows/deploy.yml` | Only on push to `main`. Reads GitHub Actions secrets (`AWS_*`, `VPC_ID`, `MYSQL_*`, …). Not relevant to local dev. |

---

## Wired to YOUR machine (not the tutorial's)

| Where | Tutorial's value (`.example`) | Your value (real file) |
|---|---|---|
| `application.properties` DB host default | `localhost` | **`192.168.122.132`** — your KVM/libvirt VM |
| `application.properties` DB user / pass | `CHANGE_ME` / `CHANGE_ME` | **`varun` / `varunkewlani`** |
| `application.properties` Kafka bootstrap | `${KAFKA_HOST:localhost}:${KAFKA_PORT:9092}` | **`192.168.122.132:9092`** hardcoded (no env var) |
| `application.properties` `jwt.secret` | `CHANGE_ME` | the `357638...` sample constant |
| `services.yml` kafka `KAFKA_ADVERTISED_LISTENERS` | `PLAINTEXT://127.0.0.1:9092` | If you run this compose file on the VM and connect from the host, change `127.0.0.1` → `192.168.122.132` |
| `cloudformation-template.yaml` | `mysql.myapp.local`, `kafka.myapp.local`, `t3.micro` | Internal AWS-VPC DNS names — only exist inside that stack. Ignore for local dev. |

> **No AWS Lightsail / EC2 public IP or hostname is hardcoded anywhere** in the repo.
> The one machine-specific address baked into code is `192.168.122.132` (your local VM),
> appearing in `application.properties` **twice** (DB URL default + Kafka bootstrap).
> Everything AWS-related is parameterised through GitHub Actions secrets.

**Consistency check for the two services:** the pair only works if —
`authservice` Kafka broker == `userservice` Kafka broker, **and**
`authservice spring.kafka.topic-json.name` == `userservice spring.kafka.topic-json.name` (`user_service`).
