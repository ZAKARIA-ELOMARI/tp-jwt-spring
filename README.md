# TP — Authentification JWT avec Spring Boot & Spring Security

API REST sécurisée par **JSON Web Token (JWT)** : inscription, connexion, puis accès aux
routes protégées via l'en-tête `Authorization: Bearer <token>`.
Projet réalisé dans le cadre du cours de sécurité (ENSET Mohammedia).

---

## 1. Technologies

| Outil | Version |
|---|---|
| Java | 21 |
| Spring Boot | 4.1.1 |
| Spring Security | (fournie par Spring Boot) |
| Spring Data JPA + H2 (en mémoire) | — |
| jjwt (io.jsonwebtoken) | 0.13.0 |
| Lombok | — |
| Maven | wrapper `mvnw` inclus |

---

## 2. Principe de fonctionnement

```
Client                         API Spring Boot
  │  POST /auth/register            │  enregistre l'utilisateur (mot de passe haché BCrypt)
  │────────────────────────────────>│
  │  POST /auth/login               │  vérifie email + mot de passe
  │────────────────────────────────>│  génère un JWT signé (HS256)
  │<──────── { "token": "..." } ────│
  │                                 │
  │  GET /api/hello                 │  JwtFilter : lit l'en-tête Authorization,
  │  Authorization: Bearer <token>  │  vérifie signature + expiration,
  │────────────────────────────────>│  place l'utilisateur dans le SecurityContext
  │<──────── 200 OK ────────────────│
```

---

## 3. Structure du projet

```
src/main/java/ma/enset/tp_jwt_spring
├── config
│   └── SecurityConfig.java              # BCrypt, AuthenticationManager, chaîne de filtres
├── controller
│   ├── AuthController.java              # /auth/register, /auth/login
│   └── UserController.java              # /api/hello, /api/profile (protégées)
├── dto
│   ├── AuthResponse.java                # { token }
│   ├── LoginRequest.java                # { email, password }
│   └── RegisterRequest.java             # { email, password }
├── entity
│   └── AppUser.java                     # table "users" : id, email, password, role
├── repository
│   └── UserRepository.java              # findByEmail, existsByEmail
├── security
│   └── JwtFilter.java                   # filtre exécuté à chaque requête
├── service
│   ├── CustomerUserDetailsServiceImpl.java  # AppUser -> UserDetails
│   └── JwtService.java                  # génération / lecture / validation du JWT
└── TpJwtSpringApplication.java
```

---

## 4. Configuration (`application.properties`)

```properties
spring.application.name=tp-jwt-spring

spring.datasource.url=jdbc:h2:mem:securitydbdemo
spring.datasource.driverClassName=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=

spring.jpa.show-sql=true
spring.jpa.hibernate.ddl-auto=update

spring.h2.console.enabled=true
spring.h2.console.path=/h2-console

jwt.secret=<clé HMAC encodée en Base64>
jwt.expiration=3600000        # 1 heure (en millisecondes)
```

> ⚠️ La clé `jwt.secret` du TP est une clé de démonstration. En production, elle doit être
> générée aléatoirement et fournie par variable d'environnement, jamais versionnée.

---

## 5. Lancer le projet

```bash
# Linux / macOS
./mvnw spring-boot:run

# Windows
mvnw.cmd spring-boot:run
```

L'API démarre sur **http://localhost:8080**.
La console H2 est disponible sur http://localhost:8080/h2-console
(JDBC URL : `jdbc:h2:mem:securitydbdemo`, utilisateur : `sa`, mot de passe vide).

📸 **Screenshot — démarrage de l'application**

![Démarrage de l'application](docs/screenshots/01-demarrage.png)

---

## 6. Règles de sécurité

| Route | Accès |
|---|---|
| `POST /auth/register`, `POST /auth/login` | Public |
| `/h2-console/**` | Public (développement uniquement) |
| `/api/**` | **Authentifié** (JWT valide obligatoire) |
| Autres | Public |

L'API est **stateless** (aucune session) et CSRF est désactivé.

---

## 7. Tests avec Postman

Un fichier `requests.http` est aussi fourni à la racine pour rejouer les requêtes.

### 7.1 Inscription — `POST /auth/register`

- **URL** : `http://localhost:8080/auth/register`
- **Headers** : `Content-Type: application/json`
- **Body (raw / JSON)** :

```json
{
  "email": "user@enset.ma",
  "password": "1234"
}
```

- **Réponse attendue** : `200 OK` — `Utilisateur cree`

📸 **Screenshot — requête**

![Register — requête](docs/screenshots/02-register-requete.png)

📸 **Screenshot — réponse**

![Register — réponse](docs/screenshots/03-register-reponse.png)

### 7.2 Inscription avec un email déjà utilisé

- Rejouer la même requête.
- **Réponse attendue** : `400 Bad Request` — `Email deja utilise`

📸 **Screenshot — réponse**

![Register doublon — réponse](docs/screenshots/04-register-doublon.png)

### 7.3 Vérification en base (console H2)

Requête : `SELECT * FROM USERS;` — le mot de passe doit apparaître **haché** (`$2a$10$...`).

📸 **Screenshot — console H2**

![Console H2](docs/screenshots/05-h2-users.png)

### 7.4 Connexion — `POST /auth/login`

- **URL** : `http://localhost:8080/auth/login`
- **Body (raw / JSON)** :

```json
{
  "email": "user@enset.ma",
  "password": "1234"
}
```

- **Réponse attendue** : `200 OK`

```json
{
  "token": "eyJhbGciOiJIUzI1NiJ9...."
}
```

📸 **Screenshot — requête**

![Login — requête](docs/screenshots/06-login-requete.png)

📸 **Screenshot — réponse (token JWT)**

![Login — réponse](docs/screenshots/07-login-reponse.png)

### 7.5 Connexion avec un mauvais mot de passe

- **Réponse attendue** : échec d'authentification (pas de token délivré).

📸 **Screenshot — réponse**

![Login échec](docs/screenshots/08-login-echec.png)

### 7.6 Accès à une route protégée **sans** token — `GET /api/hello`

- **Réponse attendue** : `403 Forbidden` (ou `401`) — accès refusé.

📸 **Screenshot — réponse**

![Accès sans token](docs/screenshots/09-hello-sans-token.png)

### 7.7 Accès à une route protégée **avec** token — `GET /api/hello`

- **URL** : `http://localhost:8080/api/hello`
- **Onglet Authorization** : type *Bearer Token*, coller le token reçu au login
  (équivalent à l'en-tête `Authorization: Bearer <token>`).
- **Réponse attendue** : `200 OK` — `Bienvenue dans l'API securisee`

📸 **Screenshot — requête (en-tête Authorization)**

![Hello — requête](docs/screenshots/10-hello-requete.png)

📸 **Screenshot — réponse**

![Hello — réponse](docs/screenshots/11-hello-reponse.png)

### 7.8 Profil de l'utilisateur connecté — `GET /api/profile`

- Même en-tête `Authorization: Bearer <token>`.
- **Réponse attendue** : `200 OK` — `Utilisateur connecte : user@enset.ma`

📸 **Screenshot — réponse**

![Profile — réponse](docs/screenshots/12-profile-reponse.png)

### 7.9 Token invalide ou modifié

- Modifier un caractère du token, ou utiliser un token expiré (`jwt.expiration` court).
- **Réponse attendue** : accès refusé (le `JwtFilter` n'authentifie pas la requête).

📸 **Screenshot — réponse**

![Token invalide](docs/screenshots/13-token-invalide.png)

### 7.10 Décodage du token (jwt.io)

Coller le token sur https://jwt.io : on retrouve le **header** (`alg: HS256`) et le
**payload** (`sub` = email, `iat`, `exp`). Le mot de passe n'y figure jamais.

📸 **Screenshot — jwt.io**

![Décodage jwt.io](docs/screenshots/14-jwt-io.png)

---

## 8. Détail des composants

- **`AppUser`** — entité JPA (table `users`), email unique, mot de passe haché, rôle `USER`.
- **`CustomerUserDetailsServiceImpl`** — charge l'utilisateur par email et le convertit en `UserDetails` Spring Security.
- **`JwtService`** — `generateToken`, `extractUsername`, `extractAllClaims`, `isTokenValid(token, userDetails)` ; clé HMAC décodée depuis le secret Base64.
- **`JwtFilter`** (`OncePerRequestFilter`) — extrait le Bearer token, le valide et renseigne le `SecurityContext`.
- **`SecurityConfig`** — `BCryptPasswordEncoder`, `AuthenticationManager`, politique `STATELESS`, `JwtFilter` placé avant `UsernamePasswordAuthenticationFilter`.
- **`AuthController`** — `/auth/register` (hachage + sauvegarde) et `/auth/login` (authentification + génération du JWT).
- **`UserController`** — endpoints protégés `/api/hello` et `/api/profile`.
