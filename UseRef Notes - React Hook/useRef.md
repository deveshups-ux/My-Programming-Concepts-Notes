# useRef — Quick Notes

## Definition (Professional)
`useRef` ek React hook hai jo ek mutable object (`{ current: value }`) return karta hai jo re-renders ke beech persist karta hai. `.current` update karne se component re-render **nahi** hota, aur ye DOM element ka reference ya koi bhi mutable value store karne ke liye use hota hai.

```js
const myRef = useRef(initialValue);
myRef.current            // read
myRef.current = newValue // update (koi setter function nahi)
```

---

## useState vs useRef

| | useState | useRef |
|---|---|---|
| Change hone pe re-render | Haan | Nahi |
| Use hota hai | UI pe dikhne wali cheez ke liye | Internal/silent cheez ke liye |
| Update syntax | `setValue(x)` | `ref.current = x` |

**Quick check:** Value screen pe dikhni chahiye → `useState`. Value bas internal hai (DOM node, timer id) → `useRef`.

---

## Main Use-Cases

**1. DOM element access karna**
```jsx
const inputRef = useRef(null);
useEffect(() => { inputRef.current?.focus(); }, []);
<input ref={inputRef} />
```

**2. Outside click detect karna (dropdown/popup band karna)**
```jsx
const popupRef = useRef(null);
useEffect(() => {
  const handler = (e) => {
    if (popupRef.current && !popupRef.current.contains(e.target)) setOpen(false);
  };
  document.addEventListener("mousedown", handler);
  return () => document.removeEventListener("mousedown", handler);
}, []);
<div ref={popupRef}>...</div>
```

**3. Timer/Interval ID store karna**
```jsx
const timerRef = useRef(null);
timerRef.current = setInterval(fn, 1000);
clearInterval(timerRef.current);
```

**4. Uncontrolled input** (value read karna bina har keystroke pe re-render kiye)
```jsx
const nameRef = useRef(null);
<input ref={nameRef} />
console.log(nameRef.current.value);
```

**5. Auto-scroll karna**
```jsx
bottomRef.current?.scrollIntoView({ behavior: "smooth" });
```

---

## Popup Example — Breakdown

- `popupRef.current` → jab tak `<div>` render nahi hota, `null`; render hote hi uska address
- `popupRef.current &&` → safety check, taki `null` pe `.contains()` call na ho
- `!...contains(e.target)` → true agar click **bahar** hua
- `addEventListener` (useEffect ke andar) → component mount hote hi listener lagao
- `return () => removeEventListener` → **cleanup**, unmount pe listener hatao (memory leak se bachne ke liye)
- `[]` → ye setup sirf ek baar chalao

---

## Common Mistakes

- UI mein dikhne wali value ke liye `useRef` use karna (re-render nahi hoga → bug)
- `useEffect` mein `[]` bhool jaana → listener har render pe dobara lagega
- Cleanup `return` bhool jaana → memory leak

---

## Good to Know (Advanced)

- **Custom hook:** outside-click logic ko `useOnClickOutside(ref, callback)` mein nikal ke reusable bana sakte hai
- **`forwardRef`:** custom component ko `ref` dene ke liye zaruri hai, kyunki `ref` normal prop ki tarah pass nahi hota
- **Re-render kyu nahi:** `.current` badalna ek plain JS object mutation hai — React isko state ki tarah track nahi karta

---

## Yaad Rakhne Wali Line

> **useRef ek mutable dabba deta hai jo re-renders ke beech bana rehta hai bina re-render trigger kiye — DOM access ya UI se related na hone wali values store karne ke liye use hota hai.**
