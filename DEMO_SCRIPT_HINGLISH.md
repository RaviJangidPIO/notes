# Demo Script — License Project (Bolne ke liye, Hinglish)

> Ye script demo ke time bolne ke liye hai. Har section ko paragraph form mein likha gaya hai taaki aap natural tarike se bol sakein. Neeche pehle ek chhota "opening" hai, phir "full flow", aur end mein "closing lines".

---

## Opening (introduction ke liye)

"Namaste sabko. Aaj main aapko dikhane wala hoon ek project jiska naam hai **AS400 License Management System**, jo humne Infoconnect ke liye banaya hai. Iska basic idea ye hai ki hamare paas kuch integration products hain — jaise **Kafka, Hub aur MuleSoft** — aur har customer ko in products ko use karne ke liye ek **valid license** chahiye. Ye system exactly wahi kaam karta hai: ye customers ke liye **secure, encrypted license files generate karta hai**, unhe store karta hai, email karta hai, aur unki **expiry ko automatically track** karta hai. Poore system ke do main hisse hain — ek **web dashboard** jise operator use karta hai, aur ek **backend** jo saara kaam peeche se handle karta hai."

---

## Full Flow (main demo, paragraph form mein)

"Toh chaliye main aapko poora flow start se end tak dikhata hoon.

Sabse pehle jab main application kholta hoon, mujhe ek **login screen** dikhti hai. Main apna email aur password daalta hoon aur login karta hoon. Peeche jo hota hai wo ye ki ye request hamare backend ko jaati hai, jahan password ko **securely BCrypt se verify** kiya jaata hai, aur agar sab sahi hai to backend ek **JWT token** generate karke wapas bhejta hai. Ye token browser mein safely store ho jaata hai, aur uske baad meri har request ke saath automatically attach ho jaata hai — matlab session secure rehta hai.

Login ke baad mujhe **main dashboard** dikhta hai. Yahan upar kuch **summary cards** hain jo batate hain ki total kitne active clients hain, kitne licenses valid hain, aur kitne jaldi expire hone wale hain. Neeche ek **table** hai jisme saare customers aur unke licenses ki list hai — naam, company, product, start date, end date, sab kuch ek jagah.

Ab main aapko dikhata hoon ki **naya license kaise banate hain**. Main 'Create License' button par click karta hoon, aur ek form khulta hai. Yahan main customer ki details bharta hoon — jaise naam, company, email, country, product select karta hoon (Kafka, Hub ya MuleSoft), AS400 serial number, aur license ki start aur end date. Jaise hi main submit karta hoon, backend mein bahut kuch hota hai. Pehle system **check karta hai ki ye license pehle se to exist nahi karti** — same email, serial aur product ke saath duplicate allow nahi hai. Phir ek **unique 9-digit license ID** generate hoti hai, saari details ko ek text format mein arrange kiya jaata hai, aur us data ko **RSA encryption se encrypt** kiya jaata hai — yaani license file ko koi bahar se padh nahi sakta. Ye encrypted content ek `.lic` file mein convert hoti hai.

Ye `.lic` file do jagah save hoti hai — actual file **AWS S3** cloud storage par jaati hai, aur uski saari details, jaise dates aur product, hamare **MySQL database** mein save hoti hain. Iske saath hi customer ko ek **email** bhi automatically chala jaata hai jisme license file attach hoti hai aur ek download link hoti hai.

Ab agar main koi license **download** karna chahoon, to main table mein us customer ke aage download button dabata hoon, aur system S3 se wahi `.lic` file laakar mujhe de deta hai. Isi tarah agar license ko **update ya renew** karna ho — maan lijiye end date badalni hai — to main edit karta hoon, aur system **same license ID ke saath nayi encrypted file** bana kar wapas store kar deta hai, aur customer ko renewal email chala jaata hai.

Ek important baat ye hai ki ye license kaam kaise karti hai. Jab customer ko ye `.lic` file milti hai, to wo file **AS400 machine par** use hoti hai. Wahan ek **public key** ki madad se file ko decrypt kiya jaata hai aur check kiya jaata hai ki serial number sahi hai ya nahi, product sahi hai ya nahi, aur license expire to nahi hui. Agar sab valid hai tabhi product chalta hai. Iska matlab **security dono taraf se strong hai** — hum private key se encrypt karte hain, aur client public key se validate karta hai.

Aur last mein, sabse useful feature — **automatic expiry tracking**. Hamara system har roz subah **9 baje automatically** check karta hai ki koi license expire hone wali to nahi. Agar kisi license ki expiry **30, 14, 7 ya 1 din** door hai, to customer ko pehle se **reminder email** chala jaata hai, aur jis din license expire hoti hai us din bhi ek notification jaata hai. Ye kaam manually bhi trigger kiya ja sakta hai dashboard se. Isse koi bhi customer bina warning ke expire nahi hota."

---

## Closing (end karne ke liye)

"Toh short mein, ye system **poora license lifecycle manage karta hai** — banane se lekar, encrypt karne, store karne, email karne, aur expiry track karne tak — sab ek hi jagah, secure aur automated tarike se. Frontend Angular par bana hai, backend Spring Boot par, aur storage ke liye MySQL aur AWS S3 use hota hai. Thank you — agar koi questions hain to main dikhane ke liye ready hoon."

---

## Quick Cheat-Sheet (agar koi technical question puche)

Ye points yaad rakhein — quick answers ke liye:

- **Frontend:** Angular 21, port 4200.
- **Backend:** Java + Spring Boot, port 9090.
- **Database:** MySQL (`licensetool`) — license ki details store hoti hain.
- **File storage:** AWS S3 — actual `.lic` files.
- **Email:** Gmail SMTP — created/renewal/expiry emails.
- **Encryption:** RSA (private key se encrypt, public key se validate).
- **Auth:** Login par JWT token milta hai, BCrypt se password check hota hai.
- **Products:** Kafka, Hub, MuleSoft (aur ENTERPRISE-wide license option).
- **Expiry scheduler:** Roz 9 AM — reminders 30/14/7/1 din pehle.

---

## Demo ke pehle checklist (taaki demo smooth chale)

- [ ] Backend chal raha hai (`localhost:9090`) — health check `/api/v1/health` par "okay" aana chahiye.
- [ ] Frontend chal raha hai (`localhost:4200`).
- [ ] Demo login ready hai (jaise `admin@example.com` / `AdminPass1!`).
- [ ] Ek sample license pehle se bana ke rakhein — taaki download/email live dikha sakein.
- [ ] Internet on hai (S3 + email ke liye).
