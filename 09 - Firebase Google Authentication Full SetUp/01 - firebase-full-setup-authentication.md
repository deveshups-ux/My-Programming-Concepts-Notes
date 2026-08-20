# Firebase Google Authentication — Kaido Project Notes

Ye notes step-by-step samjhate hain ki Google Firebase Authentication feature kaise bana — kaunsi file pehle banayi, kya logic hai, aur "kyu" waisa kiya.

---

## 📑 Table of Contents

1. [Poora Flow (ek line mein)](#poora-flow-ek-line-mein)
2. [Flow Diagram](#flow-diagram)
3. [Why This Order? (Dependency Chain)](#why-this-order-dependency-chain)
4. [Key Terms Glossary](#key-terms-glossary)
5. [Step 1: firebase.js (Frontend)](#step-1-frontendutilsfirebasejs--firebase-se-connection-banana)
6. [Step 2: axios.js (Frontend)](#step-2-frontendutilsaxiosjs--backend-se-baat-karne-ka-rasta)
7. [Step 3: App.jsx (Frontend)](#step-3-frontendsrcappjsx--login-button-ka-dimag)
8. [Step 4: firebase.js (Backend Admin)](#step-4-backendservicesauthconfigfirebasejs--backend-ka-firebase-admin-setup)
9. [Step 5: user.model.js](#step-5-backendservicesauthmodelsusermodeljs--database-ka-blueprint)
10. [Step 6: auth.controller.js](#step-6-backendservicesauthcontrollersauthcontrollerjs--asli-dimag-verification--save)
11. [Step 7: auth.route.js](#step-7-backendservicesauthroutesauthroutejs--address-book)
12. [Step 8: index.js (Auth Service)](#step-8-backendservicesauthindexjs--sab-kuch-jodne-wali-file)
13. [Step 9: index.js (Gateway)](#step-9-backendgatewayindexjs--ek-dwarpal-proxy)
14. [Client SDK vs Admin SDK](#client-sdk-vs-admin-sdk-ka-farak)
15. [Security Cleanup](#security-cleanup-jo-saath-mein-hua)
16. [Security Checklist](#-security-checklist)
17. [Troubleshooting Table — Errors We Faced](#-troubleshooting-table--errors-we-faced)
18. [Second Login Scenario (Existing User)](#-second-login-scenario-existing-user)
19. [Next Steps / TODO](#-next-steps--todo)

---

## Poora Flow (ek line mein)

**Button click → Google popup khula → token mila → frontend ne gateway ko bheja → gateway ne auth service ko forward kiya → auth service ne Firebase Admin se token verify karwaya → DB mein user check/create kiya → session cookie set ki → response wapas frontend tak pahuncha.**

---

## Flow Diagram

```
┌─────────────┐        1. Click "Continue with Google"
│   Browser   │────────────────────────────────────┐
│  (React UI) │                                     ▼
└─────────────┘                          ┌─────────────────────┐
       ▲                                 │  Google Popup /      │
       │                                 │  Firebase Auth       │
       │  2. ID Token (JWT)              │  (signInWithPopup)   │
       │◄────────────────────────────────└─────────────────────┘
       │
       │  3. POST /auth/login { token }
       ▼
┌──────────────┐   4. proxy /auth/*   ┌──────────────────┐
│   Gateway    │─────────────────────▶│   Auth Service    │
│ (CORS+proxy) │                      │  (Express)         │
└──────────────┘                      └─────────┬──────────┘
                                                 │
                                  5. verifyIdToken(token)
                                                 ▼
                                       ┌──────────────────────┐
                                       │ Firebase Admin SDK    │
                                       │ (checks signature,    │
                                       │  expiry, issuer)      │
                                       └─────────┬──────────────┘
                                                 │ decoded { uid, email, name, picture }
                                                 ▼
                                       ┌──────────────────────┐
                                       │  MongoDB (User model) │
                                       │  find or create user  │
                                       └─────────┬──────────────┘
                                                 │
                                  6. set session cookie (httpOnly)
                                                 ▼
                                       Response → Gateway → Browser
```

**Short version:** Browser Firebase se seedha baat karta hai login (Google popup) ke liye → token milta hai → us token ko apne backend ko bhejta hai → backend dobara us token ko Firebase Admin se verify karwaata hai → sirf backend ko database touch karne ki permission hai.

---

## Why This Order? (Dependency Chain)

Har file agli file ke liye zaroori thi — isiliye isi sequence mein banayi gayi:

| Kya pehle bana | Kyu zaroori tha agle step ke liye |
|---|---|
| Firebase Console project | Bina project ke koi config keys hi nahi milti |
| `frontend/utils/firebase.js` | `auth` aur `googleProvider` export karta hai — `App.jsx` inhi ko import karta hai |
| `frontend/utils/axios.js` | `App.jsx` ka `handleLogin` isi `api` instance se backend ko call karta hai |
| `frontend/src/App.jsx` | Token generate karke backend ko bhejta hai — backend ke bina ye sirf console.log tak simit rehta |
| `backend/.../config/firebase.js` | `verifyIdToken` chalane ke liye Admin SDK ka `app` chahiye — controller isi ko import karta hai |
| `backend/.../models/user.model.js` | Controller ko DB mein user save/find karna hai — schema pehle define honi chahiye |
| `backend/.../controllers/auth.controller.js` | Route ko ek function chahiye call karne ke liye — controller pehle likha, phir route |
| `backend/.../routes/auth.route.js` | `index.js` ko mount karne ke liye ek `router` chahiye |
| `backend/.../index.js` | Poori auth service ko boot karta hai — sab kuch isi mein jud jaata hai |
| `backend/gateway/index.js` | Frontend ko ek hi entry point dena tha — isliye sabse aakhir mein, jab auth service already ban chuki thi |

**Pattern samjho:** Pehle "connection layer" (Firebase config, dono taraf), phir "data layer" (model), phir "logic layer" (controller), phir "routing layer" (route → index → gateway). Ye ek natural bottom-up build order hai.

---

## Key Terms Glossary

| Term | Matlab (ek line mein) |
|---|---|
| **JWT (JSON Web Token)** | Ek encoded string jo identity + expiry ka proof hoti hai — server bina database check kiye bhi verify kar sakta hai ki ye kisne banaya |
| **ID Token** | Firebase ka JWT jo bolta hai "ye user Google se verified hai," `getIdToken()` se milta hai |
| **Middleware** | Express mein wo function jo request aur response ke beech chalta hai (jaise `express.json()`, `cors()`) — request ko modify/check karta hai aage badhane se pehle |
| **Proxy** | Ek service jo request ko as-it-is doosri service ko forward kar deti hai (gateway → auth service) |
| **CORS (Cross-Origin Resource Sharing)** | Browser ka security rule — by default ek origin (domain) doosre origin se data nahi maang sakta; `cors()` middleware explicitly allow karta hai kaunsa origin trusted hai |
| **httpOnly Cookie** | Cookie jo JavaScript se access nahi ho sakti (`document.cookie` se dikhegi hi nahi) — sirf browser aur server ke beech travel karti hai, XSS attacks se bachati hai |
| **sameSite: "strict"** | Cookie sirf tabhi bhejegi jab request usi site se aa rahi ho — cross-site request forgery (CSRF) se bachaav |
| **Service Account** | Ek non-human "identity" jo Firebase/Google Cloud deta hai backend ko admin-level access dene ke liye, `serviceAccountKey.json` isi ka credential hai |
| **Session Cookie** | Ek chhota unique ID jo browser mein store hoti hai, taaki server pehchan sake "ye wahi logged-in user hai" bina baar-baar poora login dobara karwaye |

---

## Step 1: `frontend/utils/firebase.js` — Firebase se connection banana

```js
import { initializeApp } from "firebase/app";
import { getAuth, GoogleAuthProvider } from 'firebase/auth'

const firebaseConfig = {
  apiKey: import.meta.env.VITE_FIREBASE_API_KEY,
  authDomain: "kaido-26e42.firebaseapp.com",
  projectId: "kaido-26e42",
  storageBucket: "kaido-26e42.firebasestorage.app",
  messagingSenderId: "614469988451",
  appId: "1:614469988451:web:bcfb443fc594261f324856",
};

const app = initializeApp(firebaseConfig);
export const auth = getAuth(app);
export const googleProvider = new GoogleAuthProvider();
```

**Logic:** Firebase ek third-party service hai. Tumhara app usse "baat" karne se pehle usko batana padta hai "main kaun hu, mera project konsa hai." Yahi `firebaseConfig` object karta hai — Firebase project ka **address card**.

- `initializeApp(firebaseConfig)` → Firebase SDK ko boot karta hai, ek connection object (`app`) deta hai
- `getAuth(app)` → Us connection se **authentication module** nikalta hai (login/signup handle karta hai)
- `GoogleAuthProvider` → Batata hai "main Google se login karwana chahta hu"

`apiKey` `.env` se liya (semi-public hai), baaki config values hardcoded hain kyunki public info hi hai — inse nuksan nahi hota agar public ho.

**Ye file sabse pehle kyu?** Kyunki `auth` aur `googleProvider` iske bina exist hi nahi karte — aage ki saari login files inhi ko use karengi.

---

## Step 2: `frontend/utils/axios.js` — Backend se baat karne ka rasta

```js
import axios from "axios";

const api = axios.create({
  baseURL: import.meta.env.VITE_SERVER_URL,
  withCredentials: true,
});

export default api;
```

**Logic:** Har baar poora backend URL likhna thakau hai, isliye ek pre-configured axios instance banaya — `baseURL` already set hai.

`withCredentials: true` sabse important line hai — axios ko batata hai **cookies bhi bhejo** request ke saath. Backend jo `res.cookie("session", ...)` set karta hai, wo cookie tabhi future requests mein backend ko wapas jayegi jab ye flag true ho — warna session baar-baar toot jayega.

---

## Step 3: `frontend/src/App.jsx` — Login button ka dimag

```js
import { signInWithPopup } from "firebase/auth";
import { auth, googleProvider } from "../utils/firebase.js";
import api from "../utils/axios.js";

const App = () => {
  const handleLogin = async (token) => {
    try {
      const { data } = await api.post("/auth/login", { token });
      console.log(data);
    } catch (error) {
      console.log(error);
    }
  };

  const googleLogin = async () => {
    const data = await signInWithPopup(auth, googleProvider);
    const token = await data.user.getIdToken();
    await handleLogin(token);
  };

  return <button onClick={googleLogin}>continue with google</button>;
};
```

**Do kaam ho rahe hain:**

1. **`googleLogin()`** — Button click hote hi chalta hai
   - `signInWithPopup(auth, googleProvider)` → Google ka popup khulta hai, user apna account select karta hai. Firebase khud login UI, password check, 2FA sab handle karta hai
   - Success pe `data.user.getIdToken()` se ek **JWT token** milta hai — ye ek "proof card" hai jo bolta hai "haan ye bandaa Google se verified hai, iska UID ye hai"

2. **`handleLogin(token)`** — Proof card backend ko bheja jaata hai (`api.post("/auth/login", { token })`)

**Sawal:** Frontend khud kyu na verify + save kare?
**Jawab:** Kyunki browser mein sab kuch ho to koi bhi fake token bana ke DB mein data daal sakta hai. Isliye verification aur DB writes hamesha **backend (trusted environment)** pe hone chahiye.

---

## Step 4: `backend/services/auth/config/firebase.js` — Backend ka Firebase Admin setup

```js
import { initializeApp, cert } from "firebase-admin/app";
import serviceAccount from "../serviceAccountKey.json" with { type: "json" };

export const app = initializeApp({
  credential: cert(serviceAccount),
});
```

**Logic:** Ye Step 1 jaisa hi hai, lekin power level alag hai. Frontend wala SDK sirf login karwa sakta hai — admin rights nahi. Backend `firebase-admin` SDK use karta hai, jisko `serviceAccountKey.json` (Firebase console se download hoti private key) ke through **full admin access** milta hai.

- `cert(serviceAccount)` → "Main ye hu, meri identity ye private key prove karti hai" — master key jaisa
- `initializeApp({ credential: cert(...) })` → Admin-level connection banaya

**Backend kya kar sakta hai jo frontend nahi:**
- Kisi bhi token ko verify karna (chahe kisi ne bhi banaya ho)
- Users manually create/delete/ban karna
- Custom claims (roles) set karna

⚠️ Ye key **kabhi frontend mein nahi honi chahiye** — leak hui to koi bhi poore Firebase project ka admin ban sakta hai. Isi liye ye `.gitignore` mein hai.

---

## Client SDK vs Admin SDK ka farak

| | **Client SDK** (`firebase/*`) — Frontend | **Admin SDK** (`firebase-admin/*`) — Backend |
|---|---|---|
| Kahan use hota hai | Browser (React app) | Server (Node.js) |
| Kaise authenticate hota hai | Public `apiKey` se, user ke saamne | Private `serviceAccountKey.json` se, secret |
| Kya kar sakta hai | Login karwana, apna khud ka token lena | Kisi bhi token ko verify karna, users create/delete/ban karna |
| Trust level | Low — browser mein koi bhi dekh/manipulate kar sakta hai | High — sirf server ke paas access hai |
| File is project mein | `frontend/utils/firebase.js` | `backend/services/auth/config/firebase.js` |
| Agar leak ho jaaye | Zyada risk nahi (public key hi hai) | **Bahut bada risk** — poora project compromise ho sakta hai |

**Yaad rakhne ka tarika:** Client SDK = "reception desk" (koi bhi aa ke login try kar sakta hai). Admin SDK = "master key" (sirf trusted staff ke paas, kisi ka bhi ID verify kar sakta hai, records edit kar sakta hai).

---

## Step 5: `backend/services/auth/models/user.model.js` — Database ka blueprint

```js
const userSchema = new mongoose.Schema({
  firebaseUid: { type: String, required: true, unique: true },
  username: { type: String, required: false, unique: true },
  email: { type: String, required: true },
  name: String,
  avatar: String,
}, { timestamps: true });
```

**Logic:** Ye Mongoose schema ek form template ki tarah hai — jab bhi naya user save hoga, MongoDB check karega ki fields sahi type ki hain, required wale missing to nahi.

- `firebaseUid` → Firebase ka unique ID, isi se pata chalega "ye wahi bandaa hai jo pehle bhi aaya tha"
- `unique: true` → Do users same UID/username nahi rakh sakte
- `timestamps: true` → Mongoose khud `createdAt`/`updatedAt` add karta hai

**`username` ko `required: false` kyu kiya?** Google login ke time humare paas username nahi hota (Google sirf naam, email, photo deta hai) — required rakhte to user create hi nahi hota (jo pehle error tha).

---

## Step 6: `backend/services/auth/controllers/auth.controller.js` — Asli dimag (verification + save)

```js
import { getAuth } from "firebase-admin/auth";
import { app } from "../config/firebase.js";
import User from "../models/user.model.js";
import crypto from "crypto";

export const login = async (req, res) => {
  try {
    const { token } = req.body;
    const decoded = await getAuth(app).verifyIdToken(token);

    let user = await User.findOne({ firebase: decoded.uid });

    if (!user) {
      user = await User.create({
        firebaseUid: decoded.uid,
        email: decoded.email,
        name: decoded.name,
        avatar: decoded.picture,
      });

      const sessionId = crypto.randomUUID();
      res.cookie("session", sessionId, {
        httpOnly: true,
        secure: false,
        sameSite: "strict",
        maxAge: 1000 * 60 * 60 * 24 * 7,
      });

      return res.status(201).json({ message: "User created", user });
    }

    return res.status(200).json({ message: "User logged in", user });
  } catch (error) {
    console.error("Login error:", error);
    res.status(401).json({ error: "Invalid token" });
  }
};
```

**Line by line:**

1. `const { token } = req.body;` → Frontend se aaya token nikala
2. `getAuth(app).verifyIdToken(token)` → **Sabse critical line.** Firebase Admin SDK token ko cryptographically verify karta hai — check karta hai Google ne hi banaya, expired to nahi, tampered to nahi. Sahi hone pe `decoded` object milta hai (`uid`, `email`, `name`, `picture`)
3. `User.findOne({ firebase: decoded.uid })` → DB mein check "kya ye user pehle se hai?"
   - ⚠️ **Bug note:** Yahan `firebase` likha hai lekin model field ka naam `firebaseUid` hai — mismatch hai. Isse har login pe naya user "not found" maan liya jayega. Isko `User.findOne({ firebaseUid: decoded.uid })` karna chahiye.
4. **Naya user** → `User.create()` se DB mein save, Google se mile data se
5. **Session cookie** → `crypto.randomUUID()` se random unique ID banayi, cookie mein daali:
   - `httpOnly: true` → JS se cookie access nahi ho sakti (XSS se bachaav)
   - `sameSite: "strict"` → Sirf apni site se cookie bhejegi (CSRF se bachaav)
   - `maxAge` → 7 din baad cookie khud expire
6. **Catch block** → Kuch bhi fail ho to generic "Invalid token" bheja, real error console mein log kiya (debugging ke liye)

**Analogy:** Token = ID card. `verifyIdToken` = guard ID card check kar raha hai asli hai ya nahi. Asli hai to entry (user create/login), warna 401 reject.

---

## Step 7: `backend/services/auth/routes/auth.route.js` — Address book

```js
import express from "express";
import { login } from "../controllers/auth.controller.js";

const router = express.Router();
router.post("/login", login);

export default router;
```

**Logic:** Batata hai "jab `POST /login` request aaye, to `login` function ko call karo." Route traffic ko sahi jagah bhejta hai, actual logic controller mein hota hai.

---

## Step 8: `backend/services/auth/index.js` — Sab kuch jodne wali file

```js
import express from "express";
import connectDb from "./config/db.js";
import router from "./routes/auth.route.js";

const app = express();
app.use(express.json());
app.use("/", router);
```

- `express.json()` → Middleware — incoming request ka JSON body automatically parse karta hai, taaki `req.body.token` kaam kare
- `app.use("/", router)` → Saare routes app mein mount kiye

**Ye file poore auth service ka entry point hai** — `npm run dev` karne pe sabse pehle ye chalti hai.

---

## Step 9: `backend/gateway/index.js` — Ek dwarpal (proxy)

```js
app.use(cors({ origin: process.env.FRONTEND_URL, credentials: true }));
app.use(cookieParser());
app.use("/auth", proxy(process.env.AUTH_SERVICE));
```

**Logic:** Alag-alag services (auth, chat, agent, etc.) hain. Frontend ko har service ka alag URL yaad rakhna pade to messy ho jayega. Isliye **gateway** ek single entry point hai.

- `cors({ origin: FRONTEND_URL, credentials: true })` → Sirf tumhara frontend backend se baat kar sakta hai, cookies allow
- `proxy(AUTH_SERVICE)` → `/auth/*` request aane pe gateway silently **auth service ko forward** karta hai. Frontend ko peeche kitni services hain, pata bhi nahi chalta — sirf ek gateway URL dikhta hai

---

## Security Cleanup jo saath mein hua

- `db.js` se `console.log("URI:", process.env.MONGODB_URI)` hataya — DB credentials console mein print hona band
- `.gitignore` mein `.env` aur `serviceAccountKey.json` add kiye — secrets GitHub pe kabhi commit nahi honi chahiye

---

## 🔒 Security Checklist

Ye cheezein **kabhi bhi git mein commit nahi honi chahiye**, aur inka reason bhi yaad rakho:

| Kya secret rakhna hai | Kyu |
|---|---|
| `.env` (dono — frontend + backend) | Mongo URI, API keys — leak hui to koi bhi tumhara DB/service access kar sakta hai |
| `serviceAccountKey.json` | Ye Firebase project ka **master admin key** hai — leak hui to poora project compromise |
| Console mein token/password print karna | Screenshots, terminal logs kahin bhi share ho sakte hain — accidental leak ka common source |

**Kyu `httpOnly` aur `sameSite: "strict"` use kiya cookie mein:**
- `httpOnly: true` → JavaScript (`document.cookie`) se cookie read nahi ho sakti → agar koi malicious script bhi chal jaaye site pe (XSS), wo cookie chura nahi sakta
- `sameSite: "strict"` → Cookie sirf tabhi bhejegi jab request khud tumhari site se aayi ho → koi doosri website tumhare user ke naam pe fake request nahi bhej sakti (CSRF)

**Password/key rotate kab karna zaroori hai:**
- Agar galti se `.env` ya credentials kahin public ho gaye (chat, screenshot, GitHub commit) — turant rotate karo, chahe short time ke liye hi visible hua ho
- Agar team member project chhod raha hai
- Regular interval pe (best practice, production apps mein)

---

## 🐛 Troubleshooting Table — Errors We Faced

Ye sab real errors the jo humne is project mein face kiye — future mein koi issue aaye to pehle yahan check karo:

| Error | Wajah (Cause) | Fix |
|---|---|---|
| `SyntaxError: does not provide an export named 'getAuth'` | `firebase-admin` root package se `getAuth` import kar rahe the | `import { getAuth } from "firebase-admin/auth"` — subpath se import karo |
| `ERR_PACKAGE_PATH_NOT_EXPORTED` | `initializeApp`, `cert` ko root `firebase-admin` se import kar rahe the | `import { initializeApp, cert } from "firebase-admin/app"` |
| `MongooseError: ReplicaSetNoPrimary` | Atlas IP whitelist mein current IP nahi tha, ya cluster paused tha | Atlas → Network Access → IP add karo; cluster resume karo |
| `Failed to resolve import "axios"` | `axios` package `frontend/node_modules` mein install nahi tha | `npm install axios` frontend folder ke andar chalao |
| `POST /auth/login 401 (Unauthorized)` | Generic catch block sab errors ko "Invalid token" bata raha tha, asli error chhupa hua tha | `console.error("Login error:", error)` add karke asli error dekha |
| `ValidationError: username is required` | Google login se `username` field milta hi nahi, lekin schema mein `required: true` tha | Schema mein `username: { required: false }` kiya |
| `ReferenceError: crypto is not defined` (potential) | `crypto.randomUUID()` use kiya bina import kiye | `import crypto from "crypto"` add kiya (Node 19+ mein global bhi available hai, but explicit best hai) |
| `firebase` vs `firebaseUid` field mismatch | `User.findOne({ firebase: decoded.uid })` likha, lekin model mein field `firebaseUid` hai | Query ko `User.findOne({ firebaseUid: decoded.uid })` karna hai (**pending fix**) |

---

## 🔁 Second Login Scenario (Existing User)

Notes mein abhi tak sirf **naya user** wala case detail mein cover hua hai. Dusri baar login karne pe kya hota hai:

```js
if (!user) {
  // naya user create + session cookie set
} else {
  return res.status(200).json({ message: "User logged in", user });
  // ⚠️ yahan session cookie set nahi ho rahi!
}
```

**Issue:** Jab user **pehli baar** login karta hai, session cookie set hoti hai. Lekin jab wahi user **dusri baar** login karta hai (already exists), sirf `200` response bhej diya jaata hai — **cookie dobara set nahi hoti**. Agar purani cookie expire ho chuki ho ya browser change ho, to user "logged in" response ke bawajood actually session establish nahi hoga.

**Fix karna hoga:** Cookie-setting logic ko `if/else` ke bahar nikalna chahiye, taaki har successful login pe (naya ho ya purana) session cookie set ho:

```js
const decoded = await getAuth(app).verifyIdToken(token);
let user = await User.findOne({ firebaseUid: decoded.uid });
let statusCode = 200;
let message = "User logged in";

if (!user) {
  user = await User.create({
    firebaseUid: decoded.uid,
    email: decoded.email,
    name: decoded.name,
    avatar: decoded.picture,
  });
  statusCode = 201;
  message = "User created";
}

const sessionId = crypto.randomUUID();
res.cookie("session", sessionId, {
  httpOnly: true,
  secure: false,
  sameSite: "strict",
  maxAge: 1000 * 60 * 60 * 24 * 7,
});

return res.status(statusCode).json({ message, user });
```

---

## ✅ Next Steps / TODO

Ye sab abhi pending hai, priority order mein:

1. **`firebaseUid` field mismatch fix karo** — `User.findOne({ firebase: decoded.uid })` ko `User.findOne({ firebaseUid: decoded.uid })` karo (abhi ye bug hai, existing users dobara "naye" treat honge)
2. **Second login pe bhi session cookie set karo** — upar wala fix apply karo (cookie sirf naye user ke liye set ho rahi hai abhi)
3. **MongoDB Atlas password rotate karo** — pehle conversation mein accidentally expose hua tha
4. **Session ko actually track karo** — abhi `sessionId` sirf `crypto.randomUUID()` se ban raha hai but kahin store nahi ho raha (na DB, na memory) — isliye ye session verify karne ke kaam ka nahi hai abhi. Ek `Session` model banao ya JWT-based session use karo
5. **Logout route banao** — abhi sirf login hai, cookie clear karne ka route missing hai
6. **Auth middleware banao** — protected routes ke liye ek middleware jo session cookie check kare aur `req.user` set kare
7. **Token expiry handle karo** — Firebase ID tokens 1 hour mein expire hote hain, frontend mein refresh logic add karna hoga long sessions ke liye
8. **Error responses standardize karo** — abhi kahin `{ error: "..." }` hai kahin `{ message: "..." }`, ek consistent format follow karo poore backend mein