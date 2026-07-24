# License Project — Complete Flow (Hinglish Guide)

Ye document pure project ko samjhata hai — kya chal raha hai, kaise chal raha hai, aur end-to-end flow kya hai. Poora explanation Hinglish (Hindi + English) mein hai taaki easily samajh aaye.

---

## 1. Project kya hai? (Overview)

Ye ek **AS400 / IBM i License Management System** hai (Infoconnect ke liye). Iska kaam hai apne integration products (Kafka, Hub, MuleSoft) ke liye **encrypted license files (`.lic`) generate karna**, unko store karna, customers ko email bhejna, aur expiry track karna.

Do main parts hain:

| Part | Folder | Technology | Kaam |
|------|--------|------------|------|
| **Backend** | `as400-license-generator` | Java + Spring Boot | License banata hai, encrypt karta hai, S3 + MySQL mein store karta hai, email bhejta hai |
| **Frontend** | `license-generator-ui` | Angular 21 | Operator ke liye web dashboard — login, license CRUD, download, email trigger |

> **Important:** License ki actual validation is project mein NAHI hoti. Wo AS400 client product par hoti hai (public key se decrypt karke). Ye project sirf license **generate aur distribute** karta hai.

---

## 2. Architecture ka bird's-eye view

```
┌─────────────────────────┐         ┌──────────────────────────────┐
│  Angular UI             │  HTTP   │  Spring Boot Backend         │
│  localhost:4200         │ ──────► │  localhost:9090              │
│  (login, dashboard,     │ /api/*  │  /api/v1/*                   │
│   create/download)      │         │                              │
└─────────────────────────┘         └──────────────────────────────┘
                                              │
                              ┌───────────────┼───────────────┐
                              ▼               ▼               ▼
                          MySQL           AWS S3          Gmail SMTP
                       (licensetool)   (.lic files)    (emails bhejna)
```

- **Frontend** port `4200` par chalta hai.
- **Backend** port `9090` par chalta hai.
- Backend teen external cheezon se baat karta hai: **MySQL** (metadata), **AWS S3** (actual `.lic` files), aur **Gmail SMTP** (emails).

---

## 3. Backend deep dive (`as400-license-generator`)

### 3.1 Tech stack

| Cheez | Value |
|------|-------|
| Language | Java 8 |
| Framework | Spring Boot 2.3.7 |
| Build tool | Maven (`pom.xml`) |
| Port | 9090 |
| Database | MySQL (`licensetool` @ localhost:3306) |
| Security | Spring Security + JWT + BCrypt |
| Storage | AWS S3 (`license-generator-s3-pio`, region `us-east-1`) |
| Email | Spring Mail (Gmail SMTP) |
| Encryption | BouncyCastle RSA |
| Scheduling | Har roz subah 9 baje expiry check |

### 3.2 Package structure (main folders)

```
com.infoview.licensegenerator
├── LicenseGeneratorApplication.java   → App start + secrets load
├── config/        → AWS S3 client setup
├── controller/    → REST endpoints (API)
├── dto/           → Request/response objects
├── model/         → Database entities (Customer, User, etc.)
├── repository/    → Database queries (JPA)
├── security/      → JWT + Spring Security config
├── service/       → Business logic (S3, email, scheduler, validation)
└── exception/     → Error handling
```

### 3.3 Database tables

JPA automatically ye tables bana deta hai:

| Table | Entity | Kaam |
|-------|--------|------|
| `as400license_data` | `Customer` | License + customer records |
| `app_user` | `User` | Portal login users |
| `invalidated_tokens` | `InvalidatedToken` | Logout ke baad blacklist kiye JWT |

**Customer** table ke important columns: `Name`, `Company`, `Email`, `SerialNumber`, `LicStartdate`, `LicEnddate`, `LicenseId`, `LicenseFile` (S3 ka key), `ProductId`, `expiry_notified`.

### 3.4 Demo users (startup par auto-create hote hain)

Agar `app_user` table khaali ho, to ye users seed ho jaate hain:

| Email | Password |
|-------|----------|
| user1@example.com | Password123! |
| user2@example.com | Welcome123! |
| user3@example.com | DemoPass1! |
| admin@example.com | AdminPass1! |
| testuser@example.com | TestUser1! |

### 3.5 REST API endpoints (base path: `/api/v1`)

| Method | Path | Kaam |
|--------|------|------|
| GET | `/health` | Health check |
| GET | `/customers?page&size` | Saare licenses ki list (paginated) |
| GET | `/customers/{id}` | Ek customer ka data |
| POST | `/customers` | **License create** (file generate hoti hai) |
| PUT | `/customers/{id}` | **License update** (file regenerate hoti hai) |
| DELETE | `/customers/{id}` | Customer delete |
| GET | `/download/{customerId}` | S3 se `.lic` file download |
| POST | `/auth/login` | Login |
| POST | `/auth/logout` | Logout + token blacklist |
| GET | `/dashboard/stats` | Dashboard ke counts |
| POST | `/email/send` | Single email bhejna |
| POST | `/email/reminders` | Batch expiry reminders |
| GET/POST | `/license-scheduler/*` | Scheduler status, enable/disable, manual triggers |

---

## 4. License Generation ka logic (sabse important part)

Jab operator "Create License" karta hai, backend mein ye steps chalte hain (`LicenseController.java` + `LicenseUtil.java`):

1. **Validation** — required fields check + duplicate check (same `email + serial + product` allowed nahi).
2. **License ID generate** — ek 9-digit random number.
3. **Dates format** — input `yyyy-MM-dd` ko `dd/MM/yyyy` mein convert karta hai.
4. **Plaintext payload banata hai:**

```
customer.name={fullName}
customer.email={email}
customer.location={country}
as400.serial={serialNumber}
license.startdate={dd/MM/yyyy}
license.enddate={dd/MM/yyyy}
product.id={productId}
license.id={licenseId}
```

> Agar serial number `ENTERPRISE` ho, to `as400.serial` line skip ho jaati hai (enterprise-wide license).

5. **RSA se encrypt** karta hai:
   - Algorithm: `RSA/ECB/OAEPWITHSHA-256ANDMGF1PADDING` (BouncyCastle)
   - Private key classpath se aati hai: `KeyPair/privateKey`
   - Output ek Base64 string hoti hai.

6. **`.lic` file content banata hai:**

```
# IBMi {serialNumber} License (id: {licenseId})
{base64 encrypted text, 80 chars per line wrap}
```

7. **S3 par upload** — file name: `{licenseId}_as400-license.lic`
8. **MySQL mein save** — customer record, jisme `LicenseFile` = S3 key.
9. **Email bhejta hai** — "License Created" email (attachment + download link ke saath).

**Update (`PUT`) par:** Same license ID ke saath content regenerate hota hai, S3 par re-upload hota hai. Agar end date change hui to renewal email jaata hai.

### RSA keys ke baare mein

- Private key repo mein NAHI hai — usko generate karke classpath par `KeyPair/privateKey` rakhna padta hai (warna license create fail ho jayega).
- **Public key** AS400 client products ko diya jaata hai taaki wo license ko **offline decrypt aur validate** kar sakein.

---

## 5. Authentication flow (login/logout)

### Backend side

1. `POST /api/v1/auth/login` par `{ email, password }` aata hai.
2. Spring `AuthenticationManager` + `CustomUserDetailsService` email se user dhoondte hain.
3. Password **BCrypt** se verify hota hai.
4. `JwtUtil` ek JWT banata hai (HS512, subject = email, expiry = 24 ghante).
5. Response: `{ token, tokenType: "Bearer", email }`.
6. Logout par token DB mein blacklist ho jaata hai.

### Frontend side (`services/auth.ts`)

1. User login form bharta hai → `AuthService.login()` call hota hai.
2. Success par token `localStorage` mein `auth_token` key ke naam se store hota hai (`Bearer ` prefix ke saath).
3. `loggedIn$` (BehaviorSubject) `true` emit karta hai → UI login se dashboard par switch ho jaata hai.
4. `AuthInterceptor` har HTTP request mein `Authorization` header add kar deta hai.
5. Logout par `POST /auth/logout` + localStorage clear.

> **Security note:** Abhi backend mein saare business endpoints `permitAll()` hain (`SecurityConfig`), yaani JWT ke bina bhi API access ho jaati hai. Auth mainly **frontend UI** par enforce hoti hai. Ye ek known gap hai.

---

## 6. Frontend deep dive (`license-generator-ui`)

### 6.1 Tech stack

| Cheez | Value |
|------|-------|
| Framework | Angular 21.2 (standalone components) |
| UI library | Angular Material + SCSS |
| HTTP | `@angular/common/http` (interceptor ke saath) |
| Dev port | 4200 |
| SSR | Enabled (Express server, port 4000) |

### 6.2 Components

| Component | Kaam |
|-----------|------|
| `App` | Root shell — login/dashboard toggle karta hai |
| `Login` | Email/password login form |
| `CustomerList` | Main dashboard — stats, table, CRUD, download, email |
| `CustomerDialog` | License create/update/view form |
| `LoginSuccessDialog` | Login ke baad welcome popup |
| `LoadingDialog` | Create/update ke waqt spinner |

> **Note:** Koi Angular Router guards nahi hain. `app.routes.ts` khaali hai. Login hua hai ya nahi, us hisaab se root component conditionally `<app-login>` ya `<app-customer-list>` dikhata hai.

### 6.3 Services

| Service | Kaam |
|---------|------|
| `AuthService` | Login/logout, token localStorage mein manage |
| `AuthInterceptor` | Har request mein token attach |
| `CustomerService` | CRUD + download + dashboard stats |
| `EmailService` | License/reminder emails |
| `LicenseSchedulerService` | Scheduler status, enable/disable, manual trigger |
| `NotificationService` | Toast/alert messages |

### 6.4 API config (`app.config.ts`)

```typescript
export const API_HOST = 'http://localhost:9090';
export const API_ROOT = `${API_HOST}/api/v1`;
export const CUSTOMERS_ENDPOINT = `${API_ROOT}/customers`;
export const DOWNLOAD_ENDPOINT = `${API_ROOT}/download`;
export const EMAIL_ENDPOINT = `${API_ROOT}/email`;
export const LICENSE_SCHEDULER_ENDPOINT = `${API_ROOT}/license-scheduler`;
export const DASHBOARD_STATS_ENDPOINT = `${API_ROOT}/dashboard/stats`;
```

Dev mein `proxy.conf.json` `/api/*` ko `localhost:9090` par forward karta hai (jab `ng serve` chalta hai).

### 6.5 Products (UI mein options)

- `KAFKA` — Event streaming & messaging
- `HUB` — Central integration hub
- `MULESOFT` — MuleSoft enterprise integration
- Special serial value: `ENTERPRISE` (enterprise-wide license ke liye)

---

## 7. Complete End-to-End Flow (step by step)

### A. Login
```
User login form bharta hai
  → POST /api/v1/auth/login
  → BCrypt se password verify
  → JWT issue → localStorage mein store
  → UI dashboard par switch
```

### B. License Create
```
"Create License" click → CustomerDialog form
  → Validation (name, company, email, dates, product, serial)
  → POST /api/v1/customers
  → Duplicate check (email+serial+product)
  → 9-digit license ID generate
  → Plaintext payload banao
  → RSA private key se encrypt
  → .lic file format banao
  → S3 par upload ({licenseId}_as400-license.lic)
  → MySQL mein customer save
  → "License Created" email bhejo
  → UI dashboard + table refresh
```

### C. Storage
```
.lic bytes → S3 bucket (license-generator-s3-pio)
S3 key → MySQL ke LicenseFile column mein
Metadata (dates, serial, product, email) → as400license_data table
```

### D. Download
```
Download click → GET /api/v1/download/{customerId}
  → DB se customer.LicenseFile nikaalo
  → StorageService S3 se file laata hai
  → Browser .lic file save karta hai
```

### E. Update / Renewal
```
Customer edit → PUT /api/v1/customers/{id}
  → Same license ID ke saath content regenerate
  → S3 par re-upload
  → End date change hui → renewal email
```

### F. Validation (is repo ke bahar, AS400 par)
```
AS400 client .lic file padhta hai
  → Base64 block ko PUBLIC key se decrypt
  → serial, dates, product.id, license.id verify
  → enddate ko system date se compare
  → Product access allow/deny
```

### G. Expiry Monitoring (automatic)
```
Har roz subah 9:00 AM (LicenseScheduler):
  → Saare customers scan
  → daysLeft = 30/14/7/1 → warning email
  → daysLeft = 0 → expired notification email
  → expiry_notified flag respect karta hai

Manual trigger UI se bhi possible:
  → POST /license-scheduler/check-expiry-warnings
  → POST /license-scheduler/check-expired-licenses
```

### H. Manual Email Send
```
Row ke email icon par click
  → POST /api/v1/email/send { emailType: "LICENSE_EMAIL", customerId }
  → Backend S3 se .lic attach karke SMTP se bhejta hai
  → API fail ho to fallback: mailto: link khulta hai
```

---

## 8. Known Gaps / Mismatches (dhyan dene wali baatein)

1. **Kuch backend endpoints missing hain** jo frontend call karta hai:
   - `GET /api/v1/customers/expiring-soon`
   - `GET /api/v1/customers/valid-licenses`
   - `GET /api/v1/customers/expired-licenses`
   
   In ke bina dashboard ke kuch stat cards **404** de sakte hain.

2. **Dashboard stats mismatch:** Frontend `expiredLicenses` field expect karta hai, lekin backend `DashboardStatsDTO` mein sirf `activeClients`, `validLicenses`, `expiringSoon` hain.

3. **JWT enforce nahi hoti:** Saare business API endpoints `permitAll()` hain, yaani token ke bina bhi accessible hain.

4. **RSA private key repo mein nahi hai** — manually generate karke `KeyPair/privateKey` par rakhni padegi, warna license create fail hoga.

5. **License validation service is repo mein nahi hai** — wo AS400 client side par hoti hai.

6. **Scheduler UI commented out hai** (`customer-list.html` mein), par service logic login par phir bhi chalta hai.

---

## 9. Project ko run kaise karein (quick reference)

### Backend
```
cd as400-license-generator/as400license-generator-app
# secrets file ready karo (application-secrets.properties)
# KeyPair/privateKey classpath par rakho
mvn spring-boot:run
# → http://localhost:9090 par chalega
```

### Frontend
```
cd license-generator-ui
npm install
npm start
# → http://localhost:4200 par chalega (proxy backend par point karta hai)
```

---

**Summary (ek line mein):** Operator UI se login karke license banata hai → backend usko RSA se encrypt karke `.lic` file banata hai → S3 + MySQL mein store karta hai → customer ko email jaata hai → AS400 client us file ko public key se validate karta hai → backend scheduler expiry par reminder emails bhejta hai.
