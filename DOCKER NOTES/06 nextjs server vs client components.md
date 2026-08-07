# 6. Next.js — Server Component vs Client Component

> Note: Ye "Server" wala concept file 5 wale "Server" se related hai, but code
> likhne ke level pe. Server Component = sirf server pe (Vercel/Render ke machine
> pe) chalta hai. Client Component = browser me chalta hai.

## 6.1 Fark ek table me

|                            | Server Component                     | Client Component                         |
| -------------------------- | ------------------------------------ | ---------------------------------------- |
| Chalta kaha hai            | Server pe (Render/Vercel ka machine) | Browser me (user ke laptop pe)           |
| Database access            | ✅ Seedha kar sakta hai              | ❌ Nahi, API call karni padegi           |
| Button click, typing       | ❌ Nahi handle kar sakta             | ✅ Kar sakta hai                         |
| `useState`, `useEffect`    | ❌ Use nahi kar sakte                | ✅ Use kar sakte                         |
| Code browser tak jata hai? | ❌ Nahi, sirf HTML jata hai          | ✅ Haan, JS code bhi jata hai            |
| Speed                      | Fast (kam JS download)               | Thoda slow (JS download+run)             |
| Default hai Next.js me?    | ✅ Haan                              | ❌ Nahi, `"use client"` likhna padta hai |

---

## 6.2 Kab kaunsa use karna hai — decision trick

Apne aap se 2 sawal pucho:

1. **"Kya user isse click/type/interact karega?"** → Haan → **Client Component**
2. **"Kya sirf data dikhana hai, koi interaction nahi?"** → Haan → **Server
   Component** (Next.js me ye already default hai)

**Default hamesha Server Component rakho**, sirf jaha interaction chahiye wahi
`"use client"` likh ke Client Component banao.

---

## 6.3 Examples

### Server Component — purani chat history dikhana

```javascript
async function ChatHistory() {
  const messages = await db.getMessages(); // seedha DB se data
  return (
    <div>
      {messages.map((m) => (
        <p>{m.text}</p>
      ))}
    </div>
  );
}
```

Server pe chalta hai, database se data leta hai, sirf **final HTML** browser ko
bhejta hai — fast hai.

### Client Component — message input box

```javascript
"use client";
import { useState } from "react";

function MessageInput() {
  const [message, setMessage] = useState("");
  const [loading, setLoading] = useState(false);

  const sendMessage = async () => {
    setLoading(true);
    await fetch("/api/chat", {
      method: "POST",
      body: JSON.stringify({ message }),
    });
    setLoading(false);
  };

  return (
    <div>
      <input value={message} onChange={(e) => setMessage(e.target.value)} />
      <button onClick={sendMessage}>{loading ? "Sending..." : "Send"}</button>
    </div>
  );
}
```

Browser me chalta hai — typing/click ke turant real-time response chahiye.

### Poora page structure (Server + Client mix)

```javascript
// page.js - Server Component (default)
async function ChatPage() {
  const previousChats = await db.getUserChats();
  return (
    <div>
      <ChatHistory chats={previousChats} /> {/* Server Component */}
      <ChatInputBox /> {/* Client Component */}
    </div>
  );
}
```

### E-commerce example

```javascript
// Server Component — product list (DB se data, interaction nahi)
async function ProductList() {
  const products = await db.getProducts();
  return products.map((p) => <ProductCard key={p.id} {...p} />);
}
```

```javascript
"use client";
// Client Component — Add to Cart button (click + state chahiye)
function AddToCartButton({ productId }) {
  const [added, setAdded] = useState(false);
  return (
    <button
      onClick={() => {
        addToCart(productId);
        setAdded(true);
      }}
    >
      {added ? "Added ✓" : "Add to Cart"}
    </button>
  );
}
```

### Dark Mode Toggle example

```javascript
"use client";
import { useState } from "react";

function DarkModeToggle() {
  const [mode, setMode] = useState(true); // true = light, false = dark

  return (
    <div className={mode ? "bg-white" : "bg-black"}>
      <button onClick={() => setMode(!mode)}>
        {mode ? "Dark Mode Karo" : "Light Mode Karo"}
      </button>
    </div>
  );
}
```

**Optimization tip:** Jab do cheezein (background color + button text) **same
information** pe depend karti hain, to **ek hi state kaafi hai** — alag states
banane se code messy hota hai aur out-of-sync bugs aa sakte hain.

---

## 6.4 Simple analogy

**Server Component** = Kitchen se ready-made khana — order diya, kitchen ne bana
ke seedha plate serve kar diya, tumhe khud kuch banana nahi pada. **Fast, ready-
made.**

**Client Component** = Self-service counter — jaha tum khud interact karte ho
(customize karna), iske liye apne haath se "process" karna padta hai.

Real app me **dono ka mix** hota hai — jitna ho sake Server Component use karo
(fast + secure, kyuki DB credentials browser tak nahi jate), sirf jaha interaction
chahiye wahi Client Component.
