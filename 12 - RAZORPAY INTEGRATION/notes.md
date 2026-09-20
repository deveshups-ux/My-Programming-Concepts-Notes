# Razorpay Integration — Complete Notes (Kaido Project)

> Ye notes poore payment flow ko cover karte hain — kya code likha, kyun likha, agar na likhte to kya hota, aur interview mein ye kaise poocha ja sakta hai.

---

## 1. Poora Flow — Ek Nazar Mein

```
┌─────────────┐     ┌──────────────┐     ┌───────────────┐     ┌──────────────┐     ┌─────────────┐
│ User dabata │ --> │ Backend order │ --> │  Razorpay      │ --> │  Backend      │ --> │ Auth service │
│ "Upgrade"   │     │ create karta  │     │  checkout      │     │  signature    │     │ credits/plan │
│             │     │ hai (DB save) │     │  popup khulta  │     │  verify karta │     │ update karta │
│             │     │               │     │  hai, payment  │     │  hai          │     │ hai + session│
│             │     │               │     │  hoti hai      │     │               │     │ refresh      │
└─────────────┘     └──────────────┘     └───────────────┘     └──────────────┘     └─────────────┘

```

**6 files jo is poore flow mein shamil hain:**

| **FileRole**                                                |                                              |
| ----------------------------------------------------------- | -------------------------------------------- |
| `frontend/index.html`                                       | Razorpay ka checkout script load karta hai   |
| `frontend/components/BillingDrawer.jsx`                     | UI, button click, checkout popup kholna      |
| `frontend/features/createOrder.js`                          | Backend ko order-create request              |
| `frontend/features/verifyPayment.js`                        | Backend ko verify request                    |
| `backend/services/billing/config/razorpay.js`               | Razorpay client banata hai                   |
| `backend/services/billing/controller/billing.controller.js` | Order create + signature verify (asli logic) |
| `backend/services/auth/controllers/auth.controller.js`      | Payment ke baad credits/plan update          |

---

## 2. Step-by-Step — Code ke saath

### Step 1 — Checkout script load hona

**`frontend/index.html`****:**

```html
<script src="https://checkout.razorpay.com/v1/checkout.js"></script>

```

**Kya karta hai:** Ye script browser mein ek global object `window.Razorpay` bana deti hai. Iske bina `new window.Razorpay(...)` likhne se turant crash hota hai:

```
Uncaught TypeError: Razorpay is not a constructor

```

**💡 Samjhane layak baat:** Ye script **CDN se** load ho rahi hai (Razorpay ke apne server se), tumhare code mein nahi hai. Isliye agar internet na ho ya Razorpay ka server down ho, checkout kabhi khulega hi nahi — ye ek **external dependency** hai jo tumhare control mein nahi. Production mein isliye zaroori hai ki agar `window.Razorpay` undefined ho, to user ko ek friendly error dikhao ("Payment service unavailable"), crash mat hone do.

---

### Step 2 — "Upgrade" button dabana → Order create

**`BillingDrawer.jsx`****:**

```js
const handleUpgrade = async (planId) => {
  try {
    const data = await createOrder(planId);

```

**`frontend/features/createOrder.js`****:**

```js
export const createOrder = async (plan) => {
  try {
    const { data } = await api.post("/api/billing/create", { plan });
    return data;
  } catch (error) {
    console.error("Error create billing service:", error);
    return [];
  }
};

```

**Kya karta hai:** Frontend backend ko bolta hai *"is user ke liye ek naya payment order shuru karo"*. `plan` ek simple string hai (`"starter"` ya `"pro"`) — **na to amount, na credits frontend se bheja ja raha hai**. Ye bahut important hai (agle section mein detail se samjhaunga).

**Backend —** **`billing.controller.js`** **→** **`createOrder`****:**

```js
export const createOrder = async (req, res) => {
  try {
    const { plan } = req.body;
    const userId = req.headers["x-user-id"];
    const selectedPlan = PLANS[plan];
    if (!selectedPlan) {
      return res.status(404).json({ message: "plan not found" });
    }

    const order = await getRazorpay().orders.create({
      amount: selectedPlan.amount * 100,
      currency: "INR",
      receipt: `receipt-${Date.now()}`,
    });

    await Payment.create({
      userId,
      orderId: order.id,
      amount: selectedPlan.amount,
      credits: selectedPlan.credits,
      plan: selectedPlan.id,
      currency: order.currency,
      status: "created",
    });

    return res.status(200).json({ order, plan: selectedPlan });
  } catch (error) {
    console.error("[createOrder]", error);
    return res.status(500).json({ message: `create order error: ${error.message}` });
  }
};

```

**Line-by-line:**

1. `const selectedPlan = PLANS[plan]` — frontend se sirf `"starter"` naam aaya, actual `amount`/`credits` **backend ki apni** **`Plans.js`** **file** se nikale ja rahe hain. Frontend kabhi amount decide nahi karta.
2. `getRazorpay().orders.create({...})` — Razorpay ke server ko bola *"ek order banao"*. Ye ek unique `order.id` deta hai.
3. `amount: selectedPlan.amount * 100` — 💡 **Gotcha:** Razorpay hamesha **paise** mein amount leta hai, rupaye mein nahi. `₹199` → `19900`. Agar `*100` bhool jao, Razorpay ₹1.99 charge karega, ₹199 ki jagah — **100 guna kam**.
4. `Payment.create({..., status: "created"})` — apni DB mein ek record banaya "is order ka intent ye tha" — isse baad mein verify step ke waqt hume **apni DB se hi** asli amount/credits milenge, na ki dobara frontend se poochna padega.

---

### Step 3 — Razorpay checkout popup

**`BillingDrawer.jsx`****:**

```js
const options = {
  key: import.meta.env.VITE_RAZORPAY_KEY_ID,
  amount: data?.order?.amount,
  currency: data?.order?.currency,
  name: "KaidoAI",
  description: `${data?.plan?.name} Plan`,
  order_id: data?.order?.id,
  handler: async (response) => { ... },
  theme: { color: "#4F46E5" },
};
const razorpay = new window.Razorpay(options);
razorpay.open();

```

**Kya karta hai:** Step 2 se mile `order.id`/`amount`/`currency` ko Razorpay widget ko de diya. `.open()` popup laata hai jahan user card/UPI details dalta hai.

**💡 Samjhane layak baat — 2 alag "keys" ka concept:**

```
Backend .env:
  RAZORPAY_KEY_ID       → yahan bhi use hota hai (order create karne ke liye)
  RAZORPAY_KEY_SECRET   → 🔒 SIRF backend, kabhi kahin aur nahi

Frontend .env (Vite):
  VITE_RAZORPAY_KEY_ID  → yahi id, lekin "VITE_" prefix ke saath

```

Vite ka rule hai: jo bhi variable `VITE_` se shuru hota hai, wo **build ke waqt seedha JS bundle mein chala jaata hai** — matlab koi bhi user DevTools khol ke usko dekh sakta hai. Isliye:

- `key_id` public karna **safe** hai (Razorpay ne isi liye do keys banayi hain — ek public, ek secret)
- `key_secret` **kabhi bhi** `VITE_` prefix ke saath nahi hona chahiye, warna har user ka browser tumhari secret key expose kar dega, aur koi bhi fake signatures bana sakega.

---

### Step 4 — Payment complete hone ke baad → Verify

**`BillingDrawer.jsx`** **(handler):**

```js
handler: async (response) => {
  try {
    const data = await verifyPayment(response);
    console.log(data);
    const updatedUser = await getCurrentUser();
    dispatch(setUserData(updatedUser));
    onClose();
  } catch (error) {
    console.log(error);
  }
},

```

**Kya karta hai:** Razorpay khud `handler` ko call karta hai payment complete hone ke baad, ek `response` object ke saath: `{ razorpay_order_id, razorpay_payment_id, razorpay_signature }`. Ye 3 cheezein backend ko bheji jaati hain taaki backend **confirm** kar sake ki payment sach mein hui.

**💡 Samjhane layak baat:** `handler` sirf **success case** ke liye chalta hai. Agar user popup **beech mein band kar de** (cancel kare), `handler` kabhi call hi nahi hota — is case ko catch karne ke liye Razorpay ka alag option hota hai (`modal: { ondismiss: () => {...} }`), jo abhi is code mein nahi hai. Isका matlab agar user payment cancel kare, `Payment` document DB mein `status: "created"` pe hi forever atka reh jaata hai — na "failed" hota hai, na user ko koi feedback milta hai. **(Future improvement point — neeche "Open TODOs" mein hai)**

---

### Step 5 — Backend signature verify karta hai (asli security step)

**`billing.controller.js`** **→** **`verifyPayment`****:**

```js
export const verifyPayment = async (req, res) => {
  try {
    const { razorpay_order_id, razorpay_payment_id, razorpay_signature } = req.body;

    const generateSignature = crypto
      .createHmac("sha256", process.env.RAZORPAY_KEY_SECRET)
      .update(`${razorpay_order_id}|${razorpay_payment_id}`)
      .digest("hex");

    if (generateSignature !== razorpay_signature) {
      return res.status(400).json({ message: "Payment verification failed." });
    }

    const payment = await Payment.findOne({ orderId: razorpay_order_id });
    if (!payment) {
      return res.status(404).json({ message: "Payment not found" });
    }

    payment.status = "paid";
    payment.paymentId = razorpay_payment_id;
    await payment.save();

    await axios.post(`${process.env.AUTH_SERVICE}/update-plan`, {
      userId: payment.userId,
      plan: payment.plan,
      credits: payment.credits,
    });

    return res.status(200).json({ message: "payment verified" });
  } catch (error) {
    console.error("[verifyPayment]", error);
    return res.status(500).json({ message: `verified payment error: ${error.message}` });
  }
};

```

**💡 Ye poore integration ka sabse important hissa hai — dhyan se samjho:**

**Attack scenario (agar ye check na hota):** Koi bhi user, bina ek rupaya diye, seedha ye request bhej sakta tha:

```
POST /api/billing/verify
{ "razorpay_order_id": "order_xyz", "razorpay_payment_id": "fake123", "razorpay_signature": "kuch_bhi" }

```

Agar backend signature check na kare, credits turant mil jaate — bina real payment ke.

**Defense kaise kaam karta hai:**

1. Razorpay khud, payment complete hone par, `order_id` + `payment_id` ko tumhari `key_secret` se **HMAC-SHA256** encrypt karke ek signature deta hai.
2. Backend **wahi calculation khud se dobara** karta hai — apni `RAZORPAY_KEY_SECRET` use karke.
3. Dono signatures compare karta hai (`generateSignature !== razorpay_signature`).

**Trust anchor (ye kyun forge nahi ho sakta):** `key_secret` sirf 2 jagah hai — Razorpay ke apne server pe, aur tumhare backend `.env` mein. Attacker ke paas ye key nahi hai, isliye wo sahi signature **kabhi bana hi nahi sakta**, chahe wo `order_id`/`payment_id` guess bhi kar le.

**Doosri important cheez — Trust Boundary Rule:**

```js
await axios.post(`${process.env.AUTH_SERVICE}/update-plan`, {
  userId: payment.userId,      // 👈 apni DB se (Payment document)
  plan: payment.plan,          // 👈 apni DB se
  credits: payment.credits,    // 👈 apni DB se
});

```

Dhyan do — ye **`req.body`** **se kuch nahi le raha**. `userId`, `plan`, `credits` — sab `payment` object se aa raha hai, jo Step 2 mein **backend ne khud** banaya tha. Matlab chahe attacker frontend se `{credits: 99999}` bhi bhej de, backend usko sunta hi nahi — wo hamesha apni DB ke trusted data se hi kaam karta hai.

**Rule jo yaad rakhna hai:**

```
Frontend se aaya data     → sirf "identify" karne ke liye use karo (order_id, signature)
Apni DB mein pehle se hai → yehi "asli" data hai (amount, credits, plan)

```

---

### Step 6 — Auth service credits/plan update karta hai

**`auth.controller.js`** **→** **`updateUserPayment`****:**

```js
export const updateUserPayment = async (req, res) => {
  try {
    const { plan, credits, userId } = req.body;
    const user = await User.findById(userId);
    if (!user) {
      return res.status(404).json({ message: "User not found" });
    }
    user.plan = plan;
    user.credits += credits;
    user.totalCredits += credits;
    user.planExpiresAt = new Date(Date.now() + 30 * 24 * 60 * 60 * 1000);
    await user.save();

    const sessionId = await redis.get(`user-session-${user._id}`);
    if (sessionId) {
      await redis.set(
        `session-${sessionId}`,
        JSON.stringify({ userId: user._id, plan: user.plan, credits: user.credits, ... }),
        "EX",
        7 * 24 * 60 * 60,
      );
    }
    return res.status(200).json({ message: "update user payment succesfully" });
  } catch (error) {
    return res.status(500).json({ message: `update user payment error ${error}` });
  }
};

```

**Kya karta hai:** MongoDB mein user ka `plan`/`credits`/`totalCredits`/`planExpiresAt` update karta hai — **ye asli, permanent data hai**. Fir Redis mein us user ki **abhi-abhi active session** ko bhi naye data se refresh karta hai.

**💡 Samjhane layak baat — "kyun dono jagah update karna padta hai (MongoDB + Redis)":**

Tumhara `gateway`'s `protect` middleware **har request pe MongoDB nahi poochta** — performance ke liye wo Redis se session padhta hai (jo bahut fast hota hai). Toh agar sirf MongoDB update ho aur Redis na ho, user ka **naya plan turant reflect nahi hoga** (jab tak session expire na ho jaye, jo 7 din baad hoti hai). Isliye payment ke baad **dono jagah** update karna zaroori hai — MongoDB "source of truth" ke liye, Redis "fast-access cache" ke liye.

**`credits += credits`** — dhyan do ye `=` nahi, `+=` hai. Matlab agar user pehle se 100 credits rakhta tha aur `500` credits wala plan khareeda, to `user.credits` ban jayega `600`, `500` nahi. Ye ek **design choice** hai — credits "add" hote hain, "replace" nahi hote (jabki `plan` khud replace hota hai: `user.plan = plan`).

**Wapas frontend mein:** `handler` ke andar humne `getCurrentUser()` phir se call kiya (Step 4 mein) — ye `/api/me` hit karta hai, jo Redis ki **abhi-abhi refresh hui** session se data padhta hai. Isliye UI turant naya plan dikha deta hai, bina page refresh kiye.

---

## 3. Payment ka "Status" Lifecycle

```
"created"  ───(user payment complete karta hai)───▶  "paid"
    │
    └──(user checkout band kar de / payment fail ho)──▶  ❌ kahi nahi jaata!
                                                            "created" hi reh jata hai FOREVER

```

**Ye ek khula gap hai (abhi ke liye theek hai, lekin note karne layak):** Agar payment fail ho ya user cancel kare, `Payment` document DB mein `"created"` pe hi atka reh jaata hai — kabhi `"failed"` nahi banta. Production-grade system mein iske liye Razorpay ka **webhook** (`payment.failed` event) sunna padता hai, taaki aise cases bhi properly track ho.

---

## 4. Common Gotchas — Ek Jagah Sab

| **GotchaKya hota hai agar bhool jaoYaad rakhne ka tarika** |                                                           |                                                        |
| ---------------------------------------------------------- | --------------------------------------------------------- | ------------------------------------------------------ |
| Amount `*100`                                              | ₹199 ki jagah ₹1.99 charge hoga                           | "Razorpay paise mein sochta hai, rupaye mein nahi"     |
| `key_secret` VITE\_ prefix                                 | Poori secret key browser mein expose ho jayegi            | "Secret kabhi VITE\_ ke saath nahi"                    |
| Razorpay client top-level init                             | `.env` load hone se pehle crash                           | Lazy-init karo (function ke andar)                     |
| Service-to-service URL                                     | Gateway prefix (`/api/auth`) internal call mein galat hai | "Gateway prefix sirf browser-facing hai"               |
| Frontend se amount lena                                    | Attacker apni marzi ka amount bhej sakta hai              | "Amount/credits hamesha DB se, kabhi req.body se nahi" |
| MongoDB update, Redis bhool jaana                          | User ko naya plan turant nahi dikhega                     | "Session cache bhi refresh karo"                       |

---

## 5. Interview Questions — Isi Integration Pe Based

**Q1. Payment verify karte waqt tum amount/credits kaha se lete ho — request body se ya database se? Aur kyun?**

> **A:** Database se — kyunki request body (frontend se aaya data) ko attacker manipulate kar sakta hai. Hum sirf `order_id`/`payment_id`/`signature` request se lete hain (identify karne ke liye), lekin actual amount/credits/plan **apni DB ke** **`Payment`** **document** se nikalte hain, jo humne khud order-create ke waqt banaya tha. Isse ensure hota hai ki attacker kabhi bhi apni marzi ke credits claim nahi kar sakta.

**Q2. Signature verification kaise kaam karta hai, aur ye security ke liye zaroori kyun hai?**

> **A:** Razorpay `order_id + payment_id` ko humari `key_secret` se HMAC-SHA256 karke ek signature deta hai. Hum backend pe wahi calculation dobara karte hain apni key se, aur dono signatures compare karte hain. Agar match ho, matlab ye request sach mein Razorpay se aayi hai — kyunki `key_secret` sirf Razorpay aur humare backend ke paas hai, attacker ke paas nahi, isliye wo sahi signature forge nahi kar sakta.

**Q3.** **`RAZORPAY_KEY_ID`** **aur** **`RAZORPAY_KEY_SECRET`** **mein kya farak hai, aur dono kaha use hote hain?**

> **A:** `KEY_ID` public hai — frontend checkout widget mein bhi use hota hai (Razorpay ise "publishable key" jaisa treat karta hai). `KEY_SECRET` private hai — sirf backend server pe rehna chahiye, kyunki ye signature verify karne aur order create karne (authenticated calls) ke liye use hota hai. Agar `KEY_SECRET` frontend mein leak ho jaye, koi bhi fake signatures bana sakta hai.

**Q4. Agar user payment complete karne se pehle hi checkout popup band kar de, to kya hota hai tumhare system mein? Isko kaise better bana sakte ho?**

> **A:** Abhi `handler` function sirf success case handle karta hai — agar user cancel kare, koi callback trigger nahi hota, aur `Payment` document `"created"` status pe hi reh jaata hai. Ise better banane ke liye Razorpay ke `modal.ondismiss` callback ka use karke frontend pe track kar sakte hain, aur backend mein Razorpay ke webhooks (`payment.failed`, `order.paid`) subscribe karke server-side reliably payment status update kar sakte hain — webhooks isliye zaroori hote hain kyunki frontend callback kabhi miss bhi ho sakta hai (network issue, tab band hona), lekin webhook Razorpay ke server se directly aata hai, isliye zyada reliable hota hai.

**Q5. Tumne Razorpay client ko** **`let razorpayInstance = null`** **ke saath "lazy" kyun banaya, seedha top-level pe kyun nahi banaya?**

> **A:** ES Modules mein saare `import` statements file ke apne code se pehle evaluate hote hain — matlab agar `new Razorpay({key_id: process.env.X})` file ke top-level pe likha ho, wo `dotenv.config()` chalne se **pehle hi** chal sakta hai, jisse `process.env` abhi khaali hota hai aur crash ho jaata hai. Function ke andar (`getRazorpay()`) daal ke, ye tabhi chalta hai jab actually zarurat pade — us waqt tak app fully start ho chuki hoti hai aur env variables load ho chuke hote hain.

**Q6. Webhook aur checkout** **`handler`** **callback mein kya farak hai — dono hi to "payment success" batate hain?**

> **A:** `handler` callback **frontend (browser)** mein chalta hai — agar user tab band kar de, internet cut ho jaye, ya JS error aaye, ye kabhi chal hi nahi sakta, matlab backend ko pata hi nahi chalega payment hui bhi ya nahi. **Webhook** Razorpay ke server se **seedha backend** ko hit karta hai — browser pe depend nahi karta, isliye zyada reliable hai. Production systems dono use karte hain: `handler` fast UI-update ke liye, webhook ko **source of truth** ke roop mein (final confirmation ke liye).

**Q7. Agar dono services (billing + auth) alag machines/containers pe chal rahi ho, to** **`verifyPayment`** **ke andar** **`axios.post(AUTH_SERVICE + "/update-plan")`** **fail ho jaye (network issue) to kya hoga? Isko kaise handle karoge?**

> **A:** Abhi jaisa code hai, agar ye axios call fail ho jaaye, poora `verifyPayment` function `catch` block mein chala jaayega aur `500` return karega — lekin `Payment.status` already `"paid"` save ho chuka hoga DB mein. Matlab user ka paisa कट chuka hai, lekin credits update nahi hue. Isko fix karne ke tarike: (1) ek retry mechanism ya queue (jaise BullMQ) use karo jo automatically retry kare, (2) ek cron job banao jo `"paid"` status wale payments check kare jinke credits sync nahi hue, aur unhe reconcile kare, (3) idempotent design rakho auth-service ke `update-plan` endpoint ko, taaki retry karne pe duplicate credits na add ho jaaye.

---

## 6. Ek-Line Summary (Revision ke liye)

- **Amount** hamesha paise mein bhejo Razorpay ko (`*100`)
- **key_secret** kabhi frontend mein nahi, sirf backend `.env` mein
- **Signature verify** karna hi asli security hai — HMAC-SHA256 se, apni `key_secret` se dobara calculate karke
- **Amount/credits** hamesha apni DB se lo, request body se kabhi nahi (Trust Boundary)
- **MongoDB + Redis** dono update karo — DB permanent data ke liye, Redis fast session-cache ke liye
- **Webhooks** production mein zaroori hain — frontend `handler` akela reliable nahi hai
- **Lazy-init** karo third-party clients (Razorpay/Stripe/S3) ko — top-level pe kabhi nahi

---

*Notes banaye gaye Kaido project ke Part 2 billing integration review session se.*