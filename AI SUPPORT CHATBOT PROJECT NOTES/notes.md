# 🤖 AI Customer Support Chatbot — Project Notes

> Personal notes covering project purpose, architecture, key concepts, code explanations, and interview Q&A.

---

## 📌 1. Project Kya Problem Solve Karta Hai

### The Real-World Problem
Chhote/medium businesses (online store, coaching institute, service provider) ke paas:
- Customers baar-baar same sawal poochte hain (return policy, delivery time, COD available hai ya nahi)
- Sawal 24/7 aate hain, lekin business ke paas poori support team rakhne ka budget nahi hota
- Turant jawab na milne par customer doosri site pe chala jaata hai → **sale loss**

### The Solution (Ye Project)
Ek **SaaS platform** jo:
1. Business owner ko dashboard deta hai jaha wo apni knowledge (FAQs, policies, support email) likh sakta hai
2. AI (Gemini) us knowledge ko use karke customer ke sawalon ka jawab deta hai — **sirf diya gaya data use karta hai, khud se kuch invent nahi karta**
3. Ek single `<script>` tag copy-paste karke koi bhi website is chatbot ko embed kar sakti hai — **zero coding knowledge required**
4. AI 24/7 available hai, kabhi sota nahi

### Business Model Term
**SaaS (Software as a Service)** = Ek hi software/codebase jo multiple alag businesses use karte hain, apna-apna data alag rakhte hue.

```
Developer (tum) → SupportAI platform banate ho
     ↓
Business Owners → apni knowledge dashboard mein daalte hain
     ↓
End Customers → website pe chatbot se baat karte hain
```

**Interview mein kaise bolo:**
> "Maine ek multi-tenant SaaS chatbot platform banaya hai jo businesses ko ek embeddable AI-powered customer support widget deta hai. Har business apna knowledge base configure karta hai dashboard se, aur AI (Google Gemini) us data ko ground truth maan ke customer queries ka jawab deta hai — hallucination avoid karne ke liye. Widget ek single script tag se kisi bhi website mein embed ho jaata hai."

---

## 🏗️ 2. Tech Stack Overview

| Layer | Technology | Kaam |
|---|---|---|
| Frontend Framework | Next.js (App Router) | Pages + API routes dono ek hi framework mein |
| Language | TypeScript | Type safety |
| Database | MongoDB (via Mongoose) | Business settings/knowledge store karna |
| AI | Google Gemini (`@google/genai`) | Customer queries ka jawab generate karna |
| Auth | ScaleKit (OAuth) | Login/session management |



| Styling | Tailwind CSS | UI styling |
| Animation | Framer Motion (`motion/react`) | UI animations |
| Embed Widget | Vanilla JavaScript | Framework-free script jo kisi bhi site pe chale |

---

## 🧩 3. Core Concepts — Topic Wise (Code + Explanation)

### 3.1 MongoDB Connection Caching (Serverless Pattern)

**Problem jo solve ho raha hai:** Serverless environment (Vercel) mein har request ek naya function invocation ho sakta hai. Bina caching ke, har request pe naya DB connection banega → MongoDB "too many connections" error dega.

```ts
// src/lib/db.ts
let cached = global.mongoose;
if (!cached) {
  cached = global.mongoose = { conn: null, promise: null };
}

export const connectDb = async () => {
  if (cached.conn) return cached.conn;
  if (!cached.promise) {
    cached.promise = connect(mongodbUrl).then((c) => c.connection);
  }
  cached.conn = await cached.promise;
  return cached.conn;
};
```

**Kaam kaise karta hai:**
- `global.mongoose` — Node.js process ke global scope mein connection store karte hain, taaki warm invocations mein reuse ho
- `promise` cache isliye karte hain ki agar do requests same time pe aayein, dono same connection ka wait karein (race condition avoid)

**Analogy:** Baar-baar naya SIM card lene ke bajaye, ek hi number reuse karna.

**Interview Answer:**
> "Serverless functions stateless hote hain, isliye main global object mein connection cache karta hoon taaki warm starts mein connection reuse ho sake aur connection pool exhaust na ho."

---

### 3.2 OAuth Login Flow (ScaleKit)

**3 files milke poora flow banate hain:**

```ts
// Step 1: src/app/api/auth/login/route.ts
export async function GET(req: NextRequest) {
  const redirectUri = `${process.env.NEXT_PUBLIC_APP_URL}/api/auth/callback`;
  const url = scaleKit.getAuthorizationUrl(redirectUri);
  return NextResponse.redirect(url);
}
```
```ts
// Step 2: src/app/api/auth/callback/route.ts
export async function GET(req: NextRequest) {
  const code = searchParams.get("code");
  const session = await scaleKit.authenticateWithCode(code, redirectUri);
  response.cookies.set("access_token", session.accessToken, {
    httpOnly: true,
    maxAge: 24 * 60 * 60, // seconds (NOT milliseconds!)
    secure: true, // production mein true hona chahiye
    path: "/",
  });
}
```
```ts
// Step 3: src/lib/getSession.ts — har request pe verify
export async function getSession() {
  const token = session.get("access_token")?.value;
  const result = await scaleKit.validateToken(token);
  const user = await scaleKit.user.getUser(result.sub);
  return user;
}
```

**Flow (Authorization Code Flow — standard OAuth2 pattern):**
1. User `/login` pe jaata hai → identity provider (ScaleKit) ke login page pe redirect
2. Login successful → provider `?code=xyz` ke saath `callback` route pe redirect karta hai
3. Server us `code` ko provider ko wapas bhejta hai → `accessToken` milta hai (server-to-server, safe)
4. Token **httpOnly cookie** mein store hota hai

**Naya Word: httpOnly cookie**
Aisi cookie jo JavaScript (`document.cookie`) se access nahi ho sakti — sirf server padh sakta hai. Isse XSS attack mein bhi token chura nahi sakte.

**Analogy:** Jaise "Login with Google" karte ho — bilkul wahi mechanism hai.

**Interview Answer:**
> "Maine Authorization Code Flow implement kiya hai — user provider pe redirect hota hai, code milta hai, jo server-side pe accessToken se exchange hota hai. Token httpOnly cookie mein store karta hoon taaki client-side JavaScript se access na ho sake — ye XSS protection deta hai."

⚠️ **Bug jo mila:** `maxAge: 24 * 60 * 60 * 1000` galat tha — Next.js cookies `maxAge` **seconds** mein leta hai, milliseconds nahi. Isse cookie 24 ghante ki jagah ~277 din tak valid ho jaati.

---

### 3.3 Middleware / Route Protection

```ts
// src/proxy.ts
export async function proxy(req: NextRequest) {
  const session = await getSession();
  if (!session) {
    return NextResponse.redirect(`${process.env.NEXT_PUBLIC_APP_URL}`);
  }
  return NextResponse.next();
}

export const config = {
  matcher: "/dashboard/:path*",
};
```

**Kaam:** Request page render hone se **pehle hi** intercept hoti hai. `matcher` batata hai kaunse routes protect karne hain.

⚠️ **Bug jo mila:** `/embed` route matcher mein include nahi hai — koi bhi bina login iske andar ghus sakta hai, aur `session?.user?.id!` undefined ban jaata hai (non-null assertion `!` sirf TypeScript ko chup karata hai, runtime pe protect nahi karta).

**Analogy:** Society mein ek gate pe guard khada hai, doosra gate khula chhoda hai.

**Interview Answer:**
> "Middleware request ko page render hone se pehle intercept karta hai, jo per-component redirect logic likhne se zyada efficient hai. Matcher config specify karta hai kaunse routes middleware se guard hone hain — is case mein humne `/embed` route ko cover karna miss kiya tha, jo ek real security gap tha."

---

### 3.4 Server Component vs Client Component (Next.js App Router)

```tsx
// Server Component (default, async allowed, DB/cookies access)
const page = async () => {
  const session = await getSession();
  return <DashboardClient ownerId={session?.user?.id!} />;
};
```
```tsx
// Client Component
"use client";
const DashboardClient = ({ ownerId }: { ownerId: string }) => {
  const [state, setState] = useState(...); // interactivity yahan
};
```

**Pattern:** Server Component data fetch karta hai (cookies, DB) → prop ke through Client Component ko pass karta hai → Client Component interactivity (`useState`, `onClick`) handle karta hai.

**Analogy:** Waiter (server) kitchen se khana leke aata hai, tumhe (client) sirf khana khana hai, banana nahi.

**Interview Answer:**
> "Next.js App Router mein Server Components default hote hain aur server-only operations (DB calls, cookie reading) safely kar sakte hain bina client bundle size badhaye. Jahan interactivity chahiye (state, event handlers), wahan `'use client'` directive se Client Component banate hain. Main data-fetching ko server pe rakhta hoon aur usse props ke through client ko pass karta hoon."

---

### 3.5 AI Prompt Engineering — Grounded Prompting

```ts
const prompt = `
You are a professional customer support assistant for this business.
Use ONLY the information provided below to answer the customer's question.
Do NOT invent new policies, prices, or promises.

If the customer's question is completely unrelated to the information,
reply exactly with: "Please contact support."

--------------------
BUSINESS INFORMATION
--------------------
${KNOWLEDGE}
--------------------
CUSTOMER QUESTION
--------------------
${message}
`;

const ai = new GoogleGenAI({ apiKey: process.env.GEMINI_API_KEY });
const res = await ai.models.generateContent({
  model: "gemini-3.6-flash",
  contents: prompt,
});
```

**Key concepts:**
| Technique | Kyu use kiya |
|---|---|
| Role assignment ("You are a...") | AI ka tone/behavior set karta hai |
| "Use ONLY the information below" | **Grounding** — AI ko training knowledge se hatke sirf diya gaya data use karne pe force karta hai, **hallucination** rokta hai |
| "Do NOT invent policies/prices" | Business ke liye legal/financial risk avoid karta hai |
| Fixed fallback line | Predictable string jo app detect karke special UI (jaise "Talk to human") trigger kar sake |
| `----` delimiters | Prompt ke sections clearly separate karte hain, model confuse nahi hota |

**Naye Words:**
- **Hallucination** = AI ka khud se galat/fake fact bana lena
- **Grounding** = AI ko sirf diye gaye real data tak limit karna
- **Prompt Injection** = User apne message mein AI ki instructions override karne ki koshish kare (jaise "ignore previous instructions")

**Analogy:** Jaise naye employee ko bologe "sirf company rules follow karna, khud se customer ko kuch promise mat karna."

**Interview Answer:**
> "Maine grounded prompting use ki hai — system prompt mein explicitly instruct kiya hai ki model sirf provided business knowledge use kare, apni training data se facts na banaye. Ye hallucination minimize karta hai. Ek fixed fallback response bhi define kiya hai taaki unanswerable queries ek predictable string return karein jo frontend detect kar sake. Prompt injection ek known limitation hai jo mujhe pata hai — better mitigation ke liye `systemInstruction` parameter (jo user content se alag treat hota hai) use kiya ja sakta hai."

---

### 3.6 Chat API Route (`/api/chat`)

```ts
export async function POST(req: NextRequest) {
  try {
    const { ownerId, message } = await req.json();
    if (!message || !ownerId) {
      return NextResponse.json({ message: "..." }, { status: 400 });
    }
    await connectDb();
    const setting = await Settings.findOne({ ownerId });
    if (!setting) {
      return NextResponse.json({ message: "not configured" }, { status: 400 });
    }
    // ... prompt build karke Gemini call ...
    const response = NextResponse.json({ reply: res.text });
    response.headers.set("Access-Control-Allow-Origin", "*");
    return response;
  } catch (error) {
    return NextResponse.json({ message: `error ${error}` }, { status: 500 });
  }
}
```

**Flow:** Request → validation → DB se business ka data → AI ko prompt → jawab wapas.

**Status codes:**
- `400` = Client ki galti (galat/missing input)
- `500` = Server ki galti (crash, DB down, etc.)

**Interview Answer:**
> "Ye route pehle input validate karta hai, phir ownerId se business ka knowledge base MongoDB se fetch karta hai, ek grounded prompt banata hai, aur Gemini API ko call karke response return karta hai. CORS headers isliye set hain kyunki ye endpoint kisi bhi third-party website se call hoga (embed widget ke through)."

⚠️ **Security gap jo maine identify kiya:** Koi session/auth check nahi hai in DB queries se pehle. Koi bhi random `ownerId` bhej ke kisi bhi business ka data read/write kar sakta hai — ise **IDOR (Insecure Direct Object Reference)** kehte hain.

---

### 3.7 CORS aur Preflight Request (OPTIONS)

```ts
response.headers.set("Access-Control-Allow-Origin", "*");
response.headers.set("Access-Control-Allow-Methods", "POST,OPTIONS");
response.headers.set("Access-Control-Allow-Headers", "Content-Type");

export const OPTIONS = async () => {
  return NextResponse.json(null, {
    status: 200,
    headers: {
      "Access-Control-Allow-Origin": "*",
      "Access-Control-Allow-Methods": "POST,OPTIONS",
      "Access-Control-Allow-Headers": "content-Type",
    },
  });
};
```

**CORS kya hai:** Browser ka **Same-Origin Policy** by default rokta hai ki ek website (`myshop.com`) doosri website ke server (`supportai.com`) ko call kare. Ye security feature hai. Lekin humara use-case **legitimate cross-origin** hai (widget kisi bhi site pe embed hoga), isliye server explicitly allow karta hai via headers.

| Header | Matlab |
|---|---|
| `Access-Control-Allow-Origin: *` | Koi bhi domain is API ko call kar sakta hai |
| `Access-Control-Allow-Methods` | Kaunse HTTP methods allowed hain |
| `Access-Control-Allow-Headers` | Client kaunse custom headers bhej sakta hai |

**Preflight Request:** Jab browser cross-origin POST bhejta hai custom headers (`Content-Type: application/json`) ke saath, wo **pehle** ek `OPTIONS` request bhejta hai — "permission poochne" ke liye. Agar server `OPTIONS` handle na kare, real POST request kabhi jaayegi hi nahi.

```
1. Browser: OPTIONS /api/chat → "POST allowed hai mere origin se?"
2. Server: 200 OK + CORS headers → "Haan"
3. Browser: actual POST request bhejta hai
```

**Analogy:** Society mein ghar jaane se pehle guard se poochna "andar jaa sakta hoon?" — wahi cheez browser server se poochta hai.

**Interview Answer:**
> "Chatbot widget cross-origin se call hota hai isliye maine CORS headers explicitly set kiye — `Access-Control-Allow-Origin: *` taaki koi bhi website widget embed kar sake. Custom `Content-Type` header ki wajah se browser pehle ek OPTIONS preflight request bhejta hai; maine wo bhi explicitly handle kiya hai kyunki Next.js default se ise handle nahi karta. `*` wildcard sirf is public-facing endpoint ke liye use kiya hai — kisi sensitive/private-data endpoint pe main isse avoid karta."

---

### 3.8 Embeddable Widget Script (`chatBot.js`)

```js
(function () {
  const api_Url = "http://localhost:3000/api/chat"; // BUG: hardcoded
  const scriptTag = document.currentScript;
  const ownerId = scriptTag.getAttribute("data-owner-id");
  if (!ownerId) return;

  // button + chat box DOM banate hain (inline styles se)
  ...

  function addMessage(text, from) {
    const bubble = document.createElement("div");
    bubble.textContent = text; // XSS-safe, innerHTML nahi
    ...
  }

  sendBtn.onclick = async () => {
    const response = await fetch(api_Url, {
      method: "POST",
      headers: { "content-Type": "application/json" },
      body: JSON.stringify({ ownerId, message: text }),
    });
    const data = await response.json();
    ...
  };
})();
```

**Key concepts:**

| Concept | Explanation |
|---|---|
| **IIFE** `(function(){...})()` | Poora code isolated scope mein rehta hai, host website ke JS se clash nahi hota |
| `document.currentScript` | Script khud apne `<script>` tag ka reference nikaal ke `data-owner-id` attribute padh leta hai |
| `data-*` attribute | Custom HTML attribute banane ka standard tareeka |
| Inline styles (`Object.assign`) | Host website ke CSS se conflict avoid karne ke liye, external stylesheet nahi use kiya |
| `textContent` (not `innerHTML`) | **XSS prevention** — agar user `<script>` type kare message mein, ye plain text treat hoga, execute nahi hoga |
| `scrollTop = scrollHeight` | Auto-scroll to bottom trick |
| Optimistic UI update | User ka message turant dikhate hain, server response ka wait kiye bina — fast feel deta hai |

**Analogy:** Jaise WhatsApp Web ka widget kisi bhi browser mein chal jaata hai bina conflict ke.

**Interview Answer:**
> "Widget ek IIFE mein wrapped hai taaki host website ke global scope ko pollute na kare. Ye `document.currentScript` se apna `data-owner-id` padhta hai, jisse ek hi script file multiple businesses ke liye reusable ban jaati hai. Messages render karte time main `textContent` use karta hoon, `innerHTML` nahi, taaki user input se XSS na ho sake."

⚠️ **Bug jo mila:** API URL hardcoded `localhost:3000` hai — production mein kaam nahi karega. Fix: `new URL(scriptTag.src).origin` se dynamically derive karna chahiye.

---

### 3.9 Embed Page + Dynamic Script Generation

```tsx
"use client";
const EmbedClient = ({ ownerId }: { ownerId: string }) => {
  const [copied, setCopied] = useState(false);
  const embedCode = `<script
      src="${process.env.NEXT_PUBLIC_APP_URL}/chatBot.js"
      data-owner-id="${ownerId}">
</script>`;

  const copyCode = () => {
    navigator.clipboard.writeText(embedCode);
    setCopied(true);
    setTimeout(() => setCopied(false), 3000);
  };
  ...
};
```

**Naya Word: `NEXT_PUBLIC_` prefix**
Next.js mein sirf `NEXT_PUBLIC_` se shuru hone wale environment variables **browser/client-side code** mein accessible hote hain. Bina prefix wale (jaise `GEMINI_API_KEY`) sirf server pe available rehte hain — accidental leak se bachao.

**Clipboard API:** `navigator.clipboard.writeText()` se ek click mein text system clipboard mein copy ho jaata hai.

**Interview Answer:**
> "Har business ka embed code dynamically generate hota hai unke `ownerId` ke saath, taaki ek hi widget script file sabke liye reusable rahe. `NEXT_PUBLIC_` prefix wale env vars hi client bundle mein expose hote hain — isse main sensitive keys ko accidentally frontend mein leak hone se bachata hoon."

---

### 3.10 Loading/Disabled Button Pattern

```tsx
const [loading, setLoading] = useState(false);
const handleLogin = () => {
  setLoading(true);
  window.location.href = "/api/auth/login";
};

<button disabled={loading} onClick={handleLogin} className="disabled:opacity-60">
  {loading ? "Loading..." : "Login"}
</button>
```

**Kyu zaruri:** Redirect turant nahi hota (DNS lookup, network delay). Us gap mein agar user dobara click kare, duplicate requests/race condition ban sakti hai. `disabled` isse rokta hai.

**Analogy:** ATM se paisa nikalte time "Processing..." dikhta hai, dobara button dabane nahi dete.

**Interview Answer:**
> "Async actions (jaise redirect ya API call) ke doraan main button ko `disabled` karke aur loading text dikhake duplicate submissions prevent karta hoon — ye ek common UX pattern hai jo race conditions bhi avoid karta hai."

---

## 🐛 4. Bugs/Issues Found (Code Review Summary)

| # | Issue | Severity | Fix | Status |
|---|---|---|---|---|
| 1 | `/api/settings`, `/api/settings/get` mein session/auth check nahi tha — client-provided `ownerId` trust ho raha tha (IDOR) | 🔴 Critical | Server-side session (`getSession()`) se `ownerId` nikaala, client se lena band kiya | ✅ Fixed |
| 2 | `/embed` route middleware matcher mein missing tha | 🔴 Critical | `config.matcher` mein `/embed/:path*` add kiya | ✅ Fixed |
| 3 | Cookie `maxAge: 24*60*60*1000` — seconds ki jagah milliseconds diye the | 🟠 Important | `24 * 60 * 60` (seconds) kiya | ✅ Fixed |
| 4 | Cookie `secure: false` hardcoded tha | 🟠 Important | `process.env.NODE_ENV === "production"` use kiya | ✅ Fixed |
| 5 | `chatBot.js` mein hardcoded `localhost:3000` URL | 🔴 Critical (prod breaks) | `new URL(scriptTag.src).origin` se dynamic derive karna | ⏳ Pending |
| 6 | `/api/chat` par koi rate limiting nahi | 🟠 Important | Rate limiter add karna (cost/abuse control) | ⏳ Pending |
| 7 | `connectDb()` pehle bina `await` ke call ho raha tha | 🟡 Minor | `await connectDb()` | ✅ Fixed |
| 8 | Raw error object client ko bhej rahe (`chat error ${error}`) | 🟡 Minor | Generic message do, actual error server logs mein rakho | ⏳ Pending |
| 9 | Hindi/line-number debug comments production code mein reh gaye | 🟢 Cosmetic | Clean up karna | ⏳ Pending |
| 10 | Typo: "Loding..." | 🟢 Cosmetic | "Loading..." | ⏳ Pending |

**Note:** `/api/chat` mein `ownerId` client se lena zaroori hai (widget ko anonymous visitors call karte hain, unka login session nahi hota) — isliye ye "bug" nahi hai, bas ensure kiya gaya ki ye route sirf **read-only** rahe (kabhi data write/update na kare).

---

## 🎯 5. Interview Questions + Kaise Answer Do

### Q1. Apne project ke baare mein bताओ (Elevator Pitch)
**A:** "Maine ek multi-tenant SaaS AI customer support chatbot banaya hai using Next.js, MongoDB, aur Google Gemini. Business owners apni knowledge base configure karte hain dashboard se, aur ek single script tag embed karke apni website pe AI-powered chatbot laga sakte hain — jo unke diye gaye data ke basis pe hi customers ko jawab deta hai."

### Q2. Server Component vs Client Component mein farak?
**A:** "Server Components server pe render hote hain aur seedha DB/cookies access kar sakte hain, client ko JS bundle bhi kam bhejna padta hai. Client Components (`'use client'`) mein interactivity (state, event handlers) hoti hai jo sirf browser mein chal sakti hai. Main data-fetching server pe rakhta hoon aur props se client ko pass karta hoon."

### Q3. CORS kya hai aur tumne kaise handle kiya?
**A:** "CORS browser ki Same-Origin Policy ko relax karne ka mechanism hai. Chunki mera chatbot widget kisi bhi third-party website se cross-origin call karega, maine response headers mein `Access-Control-Allow-Origin: *` set kiya, aur `OPTIONS` method explicitly handle kiya kyunki browser custom headers (`Content-Type`) hone par pehle ek preflight `OPTIONS` request bhejta hai."

### Q4. Prompt engineering mein hallucination kaise control ki?
**A:** "System prompt mein explicitly instruct kiya ki model sirf diya gaya business data use kare, khud se facts na bataye. Unanswerable queries ke liye ek fixed fallback string define ki taaki wo predictable rahe aur frontend usse detect kar sake."

### Q5. Security ke angle se sabse bada risk kya tha jo tumne identify kiya?
**A:** "Mera `/api/chat` aur `/api/settings` routes client se aaya `ownerId` bina verify kiye trust kar rahe the — matlab koi bhi kisi aur business ka data access ya modify kar sakta tha (IDOR vulnerability). Fix ye hai ki `ownerId` session/token se derive karo, request body se nahi."

### Q6. httpOnly cookie kyu use ki, localStorage kyu nahi?
**A:** "httpOnly cookies JavaScript se accessible nahi hoti, isliye agar site pe XSS bhi ho jaye, attacker token nahi chura sakta. localStorage JS se fully accessible hota hai, jo XSS ke against zyada vulnerable hai."

### Q7. Serverless mein DB connection kaise manage kiya?
**A:** "Mongoose connection ko `global` object mein cache kiya taaki warm serverless invocations mein wahi connection reuse ho, har request pe naya connection na banana pade — jo MongoDB ki connection limit exhaust kar sakta tha."

### Q8. Widget script mein XSS kaise avoid kiya?
**A:** "Chat messages render karte waqt `innerHTML` ki jagah `textContent` use kiya — isse agar koi user apne message mein HTML/script tags daale, wo plain text ki tarah render honge, execute nahi honge."

### Q9. Agar scale karna ho (zyada users), kya improve karoge?
**A:** "Rate limiting add karunga `/api/chat` par (cost + abuse control), Redis caching add karunga frequently-asked knowledge ke liye, aur proper auth middleware sab API routes par lagaunga — abhi sirf `/dashboard` protect hai."

### Q10. `NEXT_PUBLIC_` prefix ka kya role hai?
**A:** "Next.js build-time pe sirf `NEXT_PUBLIC_` se shuru hone wale env variables ko client-side JS bundle mein inject karta hai. Baaki sab (jaise API keys) sirf server pe accessible rehte hain — ye accidental secret leakage se bachata hai."

---

## ✅ 6. What Went Well (Good Practices Already Followed)

- Clean Next.js App Router structure (Server/Client component split)
- Mongoose connection caching pattern sahi implement kiya
- Widget lightweight aur dependency-free hai (vanilla JS)
- `textContent` use kiya for XSS safety
- Grounded prompting approach for hallucination control
- httpOnly cookies for token storage

---

## 📚 7. Quick Glossary (Sab Naye Words Ek Jagah)

| Term | Matlab |
|---|---|
| SaaS | Ek software jo multiple businesses rent/use karte hain |
| httpOnly cookie | Cookie jo JS se access nahi ho sakti, sirf server padh sakta hai |
| IDOR | Insecure Direct Object Reference — client-provided ID trust karke kisi aur ka data access ho jaana |
| Hallucination | AI ka khud se galat/fake fact bana lena |
| Grounding | AI ko sirf diye gaye real data tak limit karna |
| Prompt Injection | User ka apne message se AI ki instructions override karne ki koshish |
| CORS | Mechanism jisse server bataye ki kaunse domains usse cross-origin call kar sakte hain |
| Preflight (OPTIONS) | Browser ki "permission-check" request jo asli request se pehle jaati hai |
| IIFE | `(function(){...})()` — isolated scope wala self-executing function |
| Optional chaining (`?.`) | Runtime pe safe null-check |
| Non-null assertion (`!`) | TypeScript ko "trust me" bolna — sirf compile-time, runtime safety nahi deta |
| Server Component | Server-only render hone wala component (DB/cookies access) |
| Client Component | Browser mein interactive component (`'use client'`) |
| Optimistic UI Update | Server response se pehle hi UI update kar dena, fast-feel ke liye |

---

*Notes prepared from full code review + concept walkthrough of the AI Customer Support Chatbot project.*
