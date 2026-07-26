# 🔑 Scalekit Notes — Authentication Setup (Hinglish)

Scalekit ka poora concept, code flow, aur common errors — apne Next.js project ke liye reference notes.

---

## 1️⃣ Scalekit Hai Kya?

**Simple bhasha mein:** Scalekit ek **third-party authentication/security company** hai jo tumhare app ke liye login ka poora kaam khud sambhalti hai — taaki tumhe khud password check karne, encrypt karne, security handle karne jaisi complex cheezein na banani padein.

> **Analogy:** Socho tumhare club mein entry ke liye ID check karni hai, lekin tumhare paas apna security guard nahi hai. Tumne ek **outside security company** (Scalekit) hire kar rakhi hai jo aake ID verify karti hai aur tumhe bata deti hai "haan ye banda genuine hai, andar jaane do."

**Ye kis type ka product hai:** Scalekit zyada **B2B / Enterprise SSO** ke liye bana hai — matlab agar tumhara app companies ko bechna hai aur unke employees ko "apni company ke login (Google Workspace/Microsoft) se hi enter karne do" jaisi feature chahiye, tab Scalekit kaam aata hai.

---

## 2️⃣ Poora Login Flow (Step-by-Step)

```
User "Login" button dabata hai
        ↓
Browser bhejta hai: /api/auth/login
        ↓
[login/route.ts] → Scalekit se ek authorization URL maangta hai
        ↓
User ko Scalekit ke hosted login page pe redirect kar diya jata hai
        ↓
User waha apna email/password (ya company SSO) se login karta hai
        ↓
Scalekit verify karta hai aur user ko wapas bhejta hai:
   /api/auth/callback?code=xyz123
        ↓
[callback/route.ts] → us code ko Scalekit ko wapas bhejta hai confirm karne ke liye
        ↓
Scalekit ek "accessToken" wapas deta hai (agar code valid tha)
        ↓
Tumhara server us token ko ek httpOnly cookie mein store karta hai
        ↓
User ab officially "logged in" hai
```

> **Analogy:** Tum club ke gate pe jaate ho (login click) → security company ke office bhejte hain (Scalekit ka login page) → wo ID check karke ek temporary pass dete hain (code) → tumhara club us pass ko confirm karta hai security company se (callback) → confirm hone pe tumhe ek wristband milta hai (cookie/token) jo bar bar ID dikhaye bina entry deta hai.

---

## 3️⃣ File-by-File Explanation

### `src/lib/scalekit.ts` — Setup / Connection File

```typescript
import { Scalekit } from "@scalekit-sdk/node";
export const scaleKit = new Scalekit(
  process.env.SCALEKIT_ENVIRONMENT_URL!,
  process.env.SCALEKIT_CLIENT_ID!,
  process.env.SCALEKIT_CLIENT_SECRET!,
);
```

**Kya kaam karta hai:** Scalekit ka ek "connection object" banata hai — jaise security company ka phone number/contact save karna, taaki baar baar likhna na pade.

**Teeno cheezein kaha se aati hain:** Scalekit dashboard se — `.env.local` file mein set karni padti hain:
- `SCALEKIT_ENVIRONMENT_URL` → tumhare Scalekit workspace ka address
- `SCALEKIT_CLIENT_ID` → tumhare app ki identity Scalekit ke paas
- `SCALEKIT_CLIENT_SECRET` → secret password (sirf server ke paas rehta hai, frontend pe kabhi expose mat karna)

⚠️ **Common Mistake:** Agar in teeno mein se koi bhi galat/khaali hai, poora flow silently fail ho jata hai — koi clear error nahi dikhta.

---

### `src/app/api/auth/login/route.ts` — Login Shuru Karne Wali File

```typescript
export async function GET(req: NextRequest) {
  const redirectUri = `${process.env.NEXT_PUBLIC_APP_URL}/api/auth/callback`;
  const url = scaleKit.getAuthorizationUrl(redirectUri);
  console.log(url);
  return NextResponse.redirect(url);
}
```

**Kya kaam karta hai:**
1. `redirectUri` banata hai — matlab Scalekit ko batata hai "verify hone ke baad user ko yaha wapas bhej dena"
2. `getAuthorizationUrl()` — Scalekit se ek special login-page URL maangta hai
3. `NextResponse.redirect(url)` — user ko us URL pe bhej deta hai

**Kab chalta hai:** Jab user "Login" button dabata hai aur browser `/api/auth/login` pe GET request bhejta hai.

⚠️ **Common Mistake:** `redirectUri` **exactly** wahi hona chahiye jo Scalekit dashboard ke "Allowed Redirect URIs" mein register kiya hai — http/https mismatch ya trailing slash bhi error de sakta hai (`invalid_redirect_uri`).

---

### `src/app/api/auth/callback/route.ts` — Verification Complete Karne Wali File

```typescript
export async function GET(req: NextRequest) {
  const { searchParams } = new URL(req.url);
  const code = searchParams.get("code");
  const redirectUri = `${process.env.NEXT_PUBLIC_APP_URL}/api/auth/callback`;

  if (!code) {
    return NextResponse.json({ message: "code is not found" }, { status: 400 });
  }

  const session = await scaleKit.authenticateWithCode(code, redirectUri);

  const response = NextResponse.redirect(`${process.env.NEXT_PUBLIC_APP_URL}`);
  response.cookies.set("access_token", session.accessToken, {
    httpOnly: true,
    maxAge: 24 * 60 * 60, // seconds mein, milliseconds nahi!
    secure: process.env.NODE_ENV === "production",
    path: "/",
  });
  return response;
}
```

**Kya kaam karta hai:**
1. Scalekit jab user ko wapas bhejta hai, URL mein ek `code` attach karke bhejta hai (`?code=xyz123`)
2. `searchParams.get("code")` us code ko nikaalta hai
3. Agar code nahi mila → error return (user ne cancel kiya ya kuch galat hua)
4. `authenticateWithCode()` — ye sabse important line hai. Ye code Scalekit ko wapas bhejta hai (server-to-server, browser involve nahi) aur verify karta hai ki ye code genuine hai
5. Verify hone pe, ek `accessToken` milta hai jo cookie mein store kar dete hain

**Kab chalta hai:** Sirf tab jab Scalekit khud is URL ko call karta hai (login complete hone ke turant baad) — user manually yaha nahi jata.

---

## 4️⃣ Common Errors & Fixes

| Error | Kyun hota hai | Fix |
|---|---|---|
| **404 on `/api/auth/login`** | Route file `app/` folder ke andar nahi hai | File hona chahiye exactly: `app/api/auth/login/route.ts` (App Router mein) |
| **`invalid_redirect_uri`** | Scalekit dashboard mein register kiya URI aur code ka `redirectUri` match nahi karta | Dono jagah exact same URL likho (http/https, trailing slash sab match karo) |
| **`code is not found`** | User ko bina `code` ke callback URL pe bhej diya gaya | Login flow properly follow ho raha hai check karo, ya user ne login cancel kiya |
| **Silent fail / undefined errors** | `.env.local` mein Scalekit ki keys missing/galat hain | `.env.local` file check karo — teeno keys sahi se set hain ya nahi |
| **Cookie expire time galat** | `maxAge` milliseconds mein diya, jabki seconds chahiye | `maxAge: 24 * 60 * 60` likho (milliseconds nahi) |
| **Production mein cookie kaam nahi kar rahi** | `secure: false` hardcoded hai | `secure: process.env.NODE_ENV === "production"` use karo |
| **`authenticateWithCode` crash ho raha hai** | Koi `try/catch` nahi hai, error unhandled reh jata hai | Callback route mein try/catch lagao taaki proper error message mile, generic 500 na aaye |

---

## 5️⃣ Scalekit Kab Use Karna Chahiye (Honest Take)

✅ **Use karo agar:**
- Tumhara app **companies ko becha** ja raha hai (B2B)
- Client company chahti hai unke employees apne office login (Google Workspace/Microsoft/Okta) se seedha tumhare app mein ghus jayein
- Enterprise-level **SSO, SAML, OIDC** support chahiye

❌ **Mat use karo agar:**
- Tumhara app **normal consumer users** (B2C) ke liye hai — simple email/Google login chahiye
- Chhota project/MVP bana rahe ho aur jaldi launch karna hai
- Iss case mein **Better Auth** ya **Supabase Auth** zyada simple aur free rahenge

---

## 6️⃣ Ek Line Mein Yaad Rakhne Wali Baat

> Scalekit = "Security company jo tumhare liye enterprise-grade login verify karti hai — tumhe khud password/security handle nahi karna padta, bas unka SDK use karo aur login/callback ki do files banao."

---

*Last updated: July 2026*
