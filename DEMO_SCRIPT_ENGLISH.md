# Demo Script — License Project (English, Speaking Guide)

> This is written for speaking out loud during the demo. The first sections are a smooth narration you can read directly. The **Create License Deep Dive** section explains exactly *what we used and how we did it*, technically.

---

## 1. Opening (introduction)

"Hi everyone. Today I'm going to walk you through the **AS400 License Management System** that we built for Infoconnect. The idea behind this project is simple: we have a few integration products — **Kafka, Hub, and MuleSoft** — and every customer needs a **valid license** to use them. This system handles that entire license lifecycle. It **generates secure, encrypted license files**, stores them safely, emails them to customers, and **automatically tracks their expiry**. The whole system has two parts — a **web dashboard** that the operator uses, and a **backend** that does all the heavy lifting behind the scenes."

---

## 2. High-Level Flow (main narration)

"Let me take you through the flow from start to finish.

When I open the application, I land on a **login screen**. I enter my email and password and log in. Behind the scenes, this request goes to our backend, where the password is **verified securely using BCrypt**, and if everything checks out, the backend generates a **JWT token** and sends it back. This token is stored safely in the browser and is automatically attached to every request after that — so the session stays secure.

After logging in, I see the **main dashboard**. At the top there are **summary cards** showing how many active clients we have, how many licenses are valid, and how many are expiring soon. Below that is a **table** listing all customers and their licenses — name, company, product, start date, end date, everything in one place.

From here I can **create a new license**, **download** an existing one, **edit or renew** a license, and **send emails** to customers. The most important piece — and the heart of this system — is the **Create License** flow, so let me explain that in detail."

---

## 3. Create License — Deep Dive (what we used and how)

> Speak this part slowly — this is where you show the real engineering.

"When I click **Create License**, a form opens where I fill in the customer details — name, company, email, country, the **product** (Kafka, Hub, or MuleSoft), the **AS400 serial number**, and the license **start and end dates**. When I submit, a whole pipeline runs on the backend. Let me explain each step and the technology we used."

### Step 1 — Request handling (Spring Boot REST API)

"The form sends a `POST` request to our endpoint `/api/v1/customers`. On the backend this is handled by a Spring Boot **REST controller** called `LicenseController`. We used **Spring Boot** for the backend because it gives us clean REST APIs, dependency injection, and easy integration with the database, cloud storage, and email — all in one framework. The incoming data is mapped into a `CustomerDTO` object and validated automatically using Java Bean Validation (`@Valid`)."

### Step 2 — Validation and duplicate prevention

"Before we generate anything, we run a **validation service** (`CustomerValidationService`). It checks that the required fields — email, serial number, product, and dates — are all present. Then it does something important: it **prevents duplicate licenses**. We query the database to check whether a license already exists with the same **email + serial number + product** combination — and this check is **case-insensitive**. If it's a duplicate, we throw a `DuplicateEntryException`, which returns an HTTP **409 Conflict**. This makes sure we never issue two identical licenses by mistake."

### Step 3 — Generate a unique license ID

"Next, we generate a **unique 9-digit license ID** using a random number generator, formatted to always be 9 digits. This ID uniquely identifies the license and is embedded both inside the file and in the file name."

### Step 4 — Format the data and build the license payload

"We then format the dates — the frontend sends dates as `yyyy-MM-dd`, and we convert them into `dd/MM/yyyy` for a clean, readable format. After that we build the actual **license content** as plain text — a set of key-value lines like this:

```
customer.name=...
customer.email=...
customer.location=...
as400.serial=...
license.startdate=...
license.enddate=...
product.id=...
license.id=...
```

There's one special case we handle: if the serial number is **`ENTERPRISE`**, we generate an **enterprise-wide license** and skip the `as400.serial` line — because an enterprise license isn't tied to one specific machine serial."

### Step 5 — Encryption (this is the security core)

"Now the most important part — **security**. We don't want anyone to be able to read or tamper with the license, so we **encrypt** the payload. We used **RSA asymmetric encryption** with the **BouncyCastle** cryptography provider. Specifically, the cipher we used is `RSA/ECB/OAEPWITHSHA-256ANDMGF1PADDING` — that's RSA with **OAEP padding using SHA-256**, which is a modern, secure padding scheme.

Here's how the key part works, and this is the clever bit:
- We **encrypt with the PRIVATE key**, which stays secret on our server. The private key is loaded from the classpath in **PKCS8** format.
- The matching **PUBLIC key** is embedded inside the AS400 client product, and it's used to **decrypt and verify** the license on the customer's machine.

So only we can *produce* a valid license (because only we have the private key), and the client can *verify* it using the public key. This guarantees the license is authentic and hasn't been forged. After encryption, the output is encoded into a **Base64 string**, and we wrap it neatly at 80 characters per line so the final file is clean and portable."

### Step 6 — Build the final `.lic` file

"We then build the final license file. It has a readable header line like:

```
# IBMi <serialNumber> License (id: <licenseId>)
```

followed by the Base64 encrypted block. This becomes our `.lic` file."

### Step 7 — Store the file (AWS S3) and metadata (MySQL)

"Now we store it in two places, and this separation is intentional:
- The **actual `.lic` file** is uploaded to **AWS S3** — our cloud storage. We used S3 because it's durable, scalable, and reliable for storing files. The file is saved with the name `<licenseId>_as400-license.lic`, with content type `application/octet-stream`.
- The **metadata** — customer details, dates, product, and importantly the **S3 file name** — is saved in our **MySQL** database using **Spring Data JPA / Hibernate**. So the database always knows which file in S3 belongs to which customer."

### Step 8 — Send the confirmation email

"Finally, once everything is saved, we automatically **send a 'License Created' email** to the customer using **Spring Mail over Gmail SMTP**. The email includes the license details and a download link. And we did this carefully — the email sending is wrapped in a try-catch, so even if the email fails for some reason, the license is still created successfully. Email failure never blocks license creation."

### One-line summary of Create License

"So to summarize the create flow: **validate → prevent duplicates → generate unique ID → build payload → RSA-encrypt with our private key → wrap into a `.lic` file → upload to S3 → save metadata in MySQL → email the customer.** Everything is secure, automated, and traceable."

---

## 4. The Rest of the Flow (quick narration)

"Beyond creating, the system supports the full lifecycle:

- **Download:** When I click download, the backend looks up the file name in the database and fetches the actual file from S3, then streams it back to the browser.
- **Update / Renewal:** If I edit a license — say I extend the end date — the system regenerates the encrypted file using the **same license ID**, re-uploads it to S3, and if the end date changed, it automatically sends a **renewal email**.
- **Validation:** The actual validation happens on the **AS400 machine** using the public key — it decrypts the file and checks the serial, product, and expiry date before allowing the product to run.
- **Automatic expiry tracking:** This is a really useful feature. A **scheduler runs every day at 9 AM**, checks all licenses, and if any license is **30, 14, 7, or 1 day** away from expiring, it sends the customer a **reminder email** in advance. On the expiry day, it sends a final notification. This can also be triggered manually from the dashboard, so no customer expires without warning."

---

## 5. Closing

"So in short, this system manages the **complete license lifecycle** — from creation, to encryption, storage, delivery, and expiry tracking — all in one secure and automated place. The frontend is built in **Angular**, the backend in **Spring Boot**, and we use **MySQL** for data, **AWS S3** for file storage, **RSA encryption** for security, and **Gmail SMTP** for emails. Thank you — I'm happy to take any questions or show any part live."

---

## 6. Technical Cheat-Sheet (for Q&A)

| Topic | What we used / How |
|-------|--------------------|
| **Frontend** | Angular 21, runs on port 4200 |
| **Backend** | Java + Spring Boot, runs on port 9090, base path `/api/v1` |
| **Database** | MySQL (`licensetool`) with Spring Data JPA / Hibernate |
| **File storage** | AWS S3 bucket, files named `<licenseId>_as400-license.lic` |
| **Email** | Spring Mail over Gmail SMTP (created / renewal / expiry emails) |
| **Encryption** | RSA with BouncyCastle, cipher `RSA/ECB/OAEPWITHSHA-256ANDMGF1PADDING`, private key in PKCS8, Base64 output |
| **Key model** | Encrypt with **private** key (server), verify with **public** key (AS400 client) |
| **Auth** | Login returns a JWT (HS512, 24h expiry); passwords hashed with BCrypt |
| **License ID** | Random 9-digit number |
| **Duplicate check** | Unique on email + serial + product (case-insensitive) → HTTP 409 if duplicate |
| **Products** | Kafka, Hub, MuleSoft (plus special `ENTERPRISE` serial for enterprise-wide licenses) |
| **Expiry scheduler** | Runs daily at 9 AM; reminders at 30 / 14 / 7 / 1 days before expiry |

---

## 7. Likely Questions & Short Answers

- **"Why RSA and not a password/AES?"** — Because we need *asymmetric* security: only our server can create a license (private key), while any client can verify it (public key) without being able to forge one.
- **"Why encrypt with the private key instead of the public key?"** — Our goal is authenticity — proving the license genuinely came from us. The client uses the embedded public key to decrypt and trust it.
- **"Why store files in S3 instead of the database?"** — S3 is built for file storage — durable, scalable, and cheap. The database only holds metadata and the S3 file reference, keeping it lightweight.
- **"What if the email fails?"** — The license is still created and saved. Email is best-effort and wrapped in error handling so it never blocks the core operation.
- **"How do you prevent duplicate licenses?"** — We check the email + serial + product combination in the database before creating, case-insensitively, and reject duplicates with a 409 error.
- **"Where does validation happen?"** — On the AS400 client product itself, using the public key — not in this system. This system generates and distributes licenses.

---

## 8. Pre-Demo Checklist

- [ ] Backend is running (`localhost:9090`) — `/api/v1/health` returns "okay".
- [ ] Frontend is running (`localhost:4200`).
- [ ] Login credentials ready (e.g. `admin@example.com` / `AdminPass1!`).
- [ ] The RSA private key is on the classpath (`KeyPair/privateKey`) — otherwise create will fail.
- [ ] One sample license already created — so you can demo download and email live.
- [ ] Internet connection is on (needed for S3 and email).
