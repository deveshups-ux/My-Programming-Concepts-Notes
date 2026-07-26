# 🔐 Authentication Notes — Libraries, Terms & Concepts (2026)

Personal notes on authentication libraries for Next.js apps, key industry terms, and how everything connects. Written for quick revision.

---

## 📊 Auth Libraries Ranking (2026)

```
Better Auth > Supabase Auth > Clerk > Scalekit > NextAuth
```

| Library | Best For | Code Effort | Notes |
|---|---|---|---|
| **Better Auth** | Full control, own your data, TypeScript-native apps | Medium-High (build own UI) | Now officially maintains Auth.js (Sept 2025). Most actively developed in 2026. Sessions live in your own Postgres DB. |
| **Supabase Auth** | Apps already using Supabase | Low | Deep integration with Row Level Security (RLS) — auth + DB authorization in one place. |
| **Clerk** | Fast launch, polished pre-built UI | Very Low | Hosted service, pay-per-MAU. Gets expensive after ~50k monthly active users. |
| **Scalekit** | B2B apps needing enterprise SSO | Medium | Hosted login page, good for company-to-company (enterprise) login needs. |
| **NextAuth (Auth.js)** | Legacy projects already using it | High | Now in maintenance/security-patch-only mode. Not recommended for new projects. |
| **WorkOS** | Big enterprise clients (SSO, SAML, SCIM, Directory Sync) | Low (via SDK + CLI) | Not a normal auth library — a full enterprise auth platform. Use only when enterprise clients demand it. |

**Quick takeaway:**
- New solo/small project → **Better Auth**
- Already on Supabase → **Supabase Auth**
- Need something live in 5 minutes with pretty UI (budget ok) → **Clerk**
- Selling to big companies who need SSO/SAML → **WorkOS** or **Scalekit**
- Old project already using NextAuth → migrate to Better Auth eventually

---

## 🧩 Key Terms Explained (Poora Explanation — Hinglish)

### 1. SaaS (Software as a Service)
**Matlab:** Koi bhi app/software jo tum apne computer mein install nahi karte, balki browser ya app ke through use karte ho aur uske liye monthly ya yearly paisa dete ho. Company apne servers pe sab kuch host karti hai, tumhe bas internet chahiye.
> **Example:** Netflix, Gmail, Notion, Canva — ye sab SaaS hain. Tum "Netflix.exe" install nahi karte, bas website/app kholte ho aur subscription pay karte ho.

### 2. B2B (Business to Business)
**Matlab:** Tumhara product ya service **normal logon (individual customers)** ko nahi, balki **doosri companies** ko becha jata hai.
> **Example:** 
> - **B2C** (Business to Consumer) = Instagram, Netflix — normal log directly use karte hain.
> - **B2B** = Slack, Salesforce — companies apne employees ke liye kharidti hain, individual log seedha nahi kharidte.

### 3. B2B SaaS
**Matlab:** Ek online software/subscription jo specifically **companies ko bechne** ke liye banaya gaya ho.
> **Example:** Notion for Teams, Zoom for Business, Slack — sab B2B SaaS products hain kyunki inhe companies apne poore team ke liye kharidti hain.

### 4. SDK (Software Development Kit)
**Matlab:** Ek "ready-made tool-box" jo koi company deti hai taaki tum unki service apne app mein easily use kar sako, bina sab kuch scratch se banaye.
> **Simple example:** Socho tumhe apni car mein music system lagana hai. Tum khud speakers, wires, sab kuch design nahi karoge — tum ek company se ek **ready-made kit** lete ho jisme sab kuch fit karne layak already banaya hua hota hai. Bas plug karo, kaam ho jaye.
> **Coding example:** `npm install @scalekit-sdk/node` — ye ek SDK hai jo Scalekit ne banaya, taaki tumhe unka poora backend system samajhne ki zaroorat na pade.

### 5. SSO (Single Sign-On)
**Matlab:** Ek hi login se **multiple apps** mein ghus jana, alag alag password yaad rakhne ki zaroorat nahi.
> **Real life example:** Socho tumhare college mein ek **ID card** hai jisse tum library, canteen, hostel — sab jagah entry kar sakte ho. Alag alag jagah alag card nahi banana padta.
> **Tech example:** Jab tum kisi website pe "**Continue with Google**" dabate ho — wo SSO hai. Ek hi Google login se tum kai apps mein ghus jate ho.
> **Company use case:** Ek company ke employees apna office ka ek hi login (jaise Microsoft/Google Workspace) use karke company ke saare tools (Slack, Notion, Zoom) mein login kar lete hain — bina har jagah alag password banaye.

### 6. Enterprise Level
**Matlab:** Chhote startups ke liye nahi, balki **bade bade companies** (jinke paas hazaaron employees hain, bahut paisa hai, security ki bahut zyada demand hai) ke liye designed features.
> **Simple example:** Ek chhoti si chai ki dukaan aur ek 5-star hotel dono khana banate hain — lekin hotel ko extra cheezein chahiye: fire safety certificate, health inspection, legal paperwork, staff training. Ye "enterprise-level" requirements hain — chhoti dukaan ko iski zaroorat nahi.
> **Coding example:** Advanced security, compliance, audit logs, multiple teams manage karna — jo ek chhota app nahi maangta, lekin Google/Microsoft jaisi badi client maangte hain.

### 7. CLI (Command Line Interface)
**Matlab:** Wo kaala screen jaha tum text likh ke commands dete ho (mouse click nahi karte) — jaise tumhara VS Code ka terminal.
> **Example:** Jab tum likhte ho `npm run dev` — ye CLI use kar rahe ho. Tumne button click nahi kiya, tumne ek command **type** karke enter dabaya.
> **"AI-powered CLI" ka matlab:** Ek smart terminal tool jo tumhare liye khud kaam kar deta hai bas ek command se — jaise `workos setup` likho, aur wo khud samajh jaye tumhara project kya hai aur sab kuch configure kar de.

### 8. SAML (Security Assertion Markup Language)
**Matlab:** Ek **purana, bahut trusted tarika** jisse companies apne employees ki identity ek doosre app ko batati hain — bina password share kiye.
> **Simple example:** Socho tumhare paas ek **sarkari attested certificate** hai jisme likha hai "ye banda XYZ company ka employee hai." Tum wo certificate kisi bhi jagah dikha do, wo verify kar lenge bina tumse phir se ID proof maange.
> SAML wahi cheez hai, lekin computer language (XML) mein — bade **corporate/government** companies isko zyada use karti hain (thoda purana tarika hai, lekin bahut trusted).

### 9. OIDC (OpenID Connect)
**Matlab:** SAML jaisa hi kaam karta hai (identity verify karna), lekin **modern, halka, aur easier** tarika hai — aaj kal ke apps isi ko prefer karte hain.
> **Simple example:** SAML = purana sarkari certificate (bhaari, complex paperwork). OIDC = aajkal ka **Digital ID card / QR code** — scan karo, verify ho jaye, fast aur simple.
> "Continue with Google" jo humne upar SSO mein dekha — wo technically **OIDC** use karta hai backend mein.

### 10. WorkOS
**Matlab:** Ek company jo **sab upar wali cheezein (SSO, SAML, OIDC, Directory Sync)** ek hi package mein deti hai, taaki tumhe khud ye sab complex cheezein banani na pade.
> **Simple example:** Socho tumhe apni shaadi organize karni hai. Tum khud caterer, decorator, photographer, DJ — sab alag se dhoondh sakte ho (mushkil, time lagega). Ya tum ek **"Wedding Planner"** hire kar lo jo sab kuch ek saath manage kar de.
> **WorkOS = Wedding Planner for enterprise login/security stuff.** Tum unko bolte ho "mujhe SSO chahiye, SAML chahiye" — wo sab backend mein handle kar dete hain, tumhe sirf unka SDK use karna hai.

### 11. OAuth
**Matlab:** Ek standard tarika jisse ek app kisi doosri app se poochta hai "iss user ki kuch details mujhe do" — bina password share kiye.
> **Simple example:** Jab tum kisi app ko bolte ho "Continue with Google," tab Google tumse poochta hai — "kya ye app tumhara naam/email dekh sakta hai?" Tum "haan" bolte ho, aur Google seedha password nahi deta, bas ek permission token deta hai. Yehi OAuth hai.
> SSO aur "Login with Google" jaisi cheezein isi OAuth protocol pe bani hoti hain.

### 12. JWT (JSON Web Token)
**Matlab:** Ek chhota sa encoded text/code jo user ki identity aur permissions store karta hai — jaise ek digital ticket.
> **Simple example:** Jaise cinema ka ticket — usme likha hota hai konsi movie, konsi seat, kab tak valid hai. Tum wo ticket dikhao, checker verify kar leta hai bina dobara counter pe jaane ke. JWT bhi aisा hi hai — server ek baar verify karta hai, phir tumhe ek "token" de deta hai jo baar baar dikhane se kaam chal jata hai.

### 13. Access Token & Refresh Token
**Access Token:** Ek short-time wala "entry pass" jo tumhe app ke andar kaam karne deta hai (usually kuch minute/ghante ke liye valid).
**Refresh Token:** Ek long-time wala "renewal pass" jo naya access token banwane ke kaam aata hai, bina user ko dobara login karwaye.
> **Simple example:** Access token = ek din ka movie ticket. Refresh token = ek membership card jisse tum roz naya ticket bina line mein lage nikal sakte ho.

### 14. Session
**Matlab:** Jab tak user login hai, uska ek "state" server ya browser mein store rehta hai — jisse pata chalta hai "ye banda already logged in hai."
> **Simple example:** Jab tum kisi dukaan mein ek baar entry check karwa lete ho aur haath pe stamp lag jata hai — tab tak dukaan ke andar tumhe baar baar ID nahi dikhani padti, stamp hi kaafi hai. Session bhi wahi stamp hai, digital form mein.

### 15. Cookie / httpOnly Cookie
**Cookie:** Browser mein store hone wala chhota sa data jo website yaad rakhti hai (jaise login status, preferences).
**httpOnly Cookie:** Ek special type ki cookie jo sirf server padh sakta hai — JavaScript (browser) usse access nahi kar sakta.
> **Simple example:** Ek normal cookie jaise diary hai jo koi bhi padh sakta hai. httpOnly cookie ek locked diary hai jiski chaabi sirf owner (server) ke paas hai — koi hacker JavaScript se chura nahi sakta.

### 16. MFA / 2FA (Multi-Factor / Two-Factor Authentication)
**Matlab:** Login ke liye sirf password kaafi nahi, ek aur "proof" bhi dena padta hai — jaise phone pe aaya OTP.
> **Simple example:** ATM se paise nikalne ke liye sirf card kaafi nahi, PIN bhi chahiye. Do cheezein milke security badhati hain — yehi MFA/2FA hai.

### 17. Passwordless Authentication
**Matlab:** Login karne ke liye password ki zaroorat hi nahi — magic link, OTP, ya fingerprint/face se login ho jata hai.
> **Simple example:** Jaise office mein fingerprint se attendance lagti hai — password yaad rakhne ki zaroorat nahi, bas apni ungli lagao.

### 18. Passkey
**Matlab:** Passwordless login ka ek modern, super-secure tarika jo tumhare device (phone/laptop) mein cryptographically stored hota hai — password ki jagah.
> **Simple example:** Jaise ghar ki chaabi copy nahi ki ja sakti bina asli chaabi ke — passkey bhi tumhare device se link hoti hai, koi doosra use nahi kar sakta, phishing se bhi safe hai.

### 19. RBAC (Role-Based Access Control)
**Matlab:** Alag alag users ko unke "role" ke hisaab se alag permissions dena.
> **Simple example:** School mein Principal, Teacher, aur Student teeno ka login alag access dega — Principal sab kuch dekh sakta hai, Teacher sirf apni class, Student sirf apna result. Yehi RBAC hai.

### 20. Vendor Lock-in
**Matlab:** Ek baar kisi company ki service use karne ke baad, use chhodna mushkil ho jana — kyunki tumhara data/system unke saath tightly juda hota hai.
> **Simple example:** Jaise ek phone company ka charger sirf unhi ke phone mein chalta hai — agar tum doosri company ka phone lo, poora charger hi badalna padega. Software mein bhi aisa hota hai — ek service (jaise Clerk) chhodna mushkil ho sakta hai agar sab kuch unke system pe depend karta ho.

### 21. MAU (Monthly Active Users)
**Matlab:** Ek mahine mein kitne users tumhare app ko actually use kar rahe hain — iske hisaab se hi zyada tar auth services (Clerk, WorkOS) apna pricing decide karti hain.
> **Simple example:** Jaise gym ka membership plan "kitne log gym aa rahe hain" ke hisaab se price badhata hai — waise hi zyada MAU hone pe auth service ka bill badh jata hai.

---

## 🔄 How a Login Flow Actually Works (OAuth/SSO Style)

```
1. User clicks "Login"
        ↓
2. App redirects user to the auth provider's (e.g. Scalekit/WorkOS) hosted login page
        ↓
3. User logs in there (provider verifies identity)
        ↓
4. Provider redirects back to your app's callback URL with a temporary "code"
        ↓
5. Your app's server sends that code back to the provider to confirm it's valid
        ↓
6. Provider returns an access token
        ↓
7. Your app stores that token in an httpOnly cookie (like a wristband)
        ↓
8. User is now "logged in" — cookie proves it on every future request
```

**Why httpOnly cookie?** JavaScript in the browser (and XSS attacks) can't read it — only the server can, which keeps the token safer.

---

## ✅ What Recruiters/Interviewers Actually Care About

They rarely care *which specific library* you used. They care about:
1. Do you understand the OAuth/SSO flow (the diagram above)?
2. Do you understand session/cookie security basics (httpOnly, token expiry, CSRF)?
3. Did you actually ship a working project (GitHub repo with real, functioning code)?

Knowing the library name looks good on a resume — but being able to *explain the flow* in an interview is what actually matters.

---

*Last updated: July 2026*