# File Upload Middleware (Multer) — Notes

## Ye hai kya?

**Naam:** File Upload Middleware (Multer-based)

> Multer ek Express middleware hai jo `multipart/form-data` requests (file uploads) handle karta hai. Maine ek `fileFilter` aur `diskStorage` configure karke ek reusable upload-middleware banaya — jo sirf PDF/Images accept karta hai, unique naam deta hai, aur 20MB size-limit lagata hai.

**File location suggestion:** `config/multer.js`

---

## Poora Final Code

```js
import fs from "fs";
import path from "path";
import multer from "multer";

const uploadDir = path.resolve("./temp");

if (!fs.existsSync(uploadDir)) {
  fs.mkdirSync(uploadDir, { recursive: true });
}

const storage = multer.diskStorage({
  destination(req, file, cb) {
    cb(null, uploadDir);
  },
  filename(req, file, cb) {
    cb(null, `${Date.now()}-${file.originalname}`);
  },
});

const fileFilter = (req, file, cb) => {
  if (file.mimetype === "application/pdf" || file.mimetype.startsWith("image/")) {
    cb(null, true);
  } else {
    cb(new Error("Only PDF and Images are allowed"));
  }
};

export default multer({
  storage,
  fileFilter,
  limits: {
    fileSize: 20 * 1024 * 1024,
  },
});

```

---

## Kaam Kaise Karta Hai — 4 Parts

### 1. Folder Taiyar Karna (server start hote hi, ek baar)

```js
if (!fs.existsSync(uploadDir)) {
  fs.mkdirSync(uploadDir, { recursive: true });
}

```

- `fs.existsSync` -> check: folder pehle se hai?
- `fs.mkdirSync(..., { recursive: true })` -> nahi hai to bana do (nested folders bhi ek saath ban jaate hain)

> Tip: `mkdirSync` (sync) yaha theek hai kyunki ye server-start pe ek hi baar chalta hai, kisi request ke beech mein nahi.

---

### 2. Storage Rules — kaha aur kis naam se save hogi

```js
destination(req, file, cb) {
  cb(null, uploadDir);          // KAHA save karni hai
}
filename(req, file, cb) {
  cb(null, `${Date.now()}-${file.originalname}`);   // KIS NAAM se
}

```

**Example:**

```
User upload karta hai: "resume.pdf"
Date.now() = 1758452891234

Final saved file: ./temp/1758452891234-resume.pdf

```

> Tip: Timestamp isliye lagaya jaata hai taaki 2 alag users ki same-naam files ek dusre ko overwrite na karein — har naam automatically unique ban jaata hai.

---

### 3. File Filter — sirf PDF/Image allow karna

```js
if (file.mimetype === "application/pdf" || file.mimetype.startsWith("image/")) {
  cb(null, true);      // allow
} else {
  cb(new Error("..."));  // reject
}

```

`file.mimetype` — har file ke saath aata hai, uska "type" batata hai:

```
PDF        -> "application/pdf"
JPG image  -> "image/jpeg"
PNG image  -> "image/png"

```

**3 example cases:**

| **FilemimetypeResult** |                          |                             |
| ---------------------- | ------------------------ | --------------------------- |
| resume.pdf             | application/pdf          | allow                       |
| photo.jpg              | image/jpeg               | allow (startsWith "image/") |
| script.exe             | application/x-msdownload | reject                      |

---

### 4. Sab Jodke Export Karna

```js
export default multer({
  storage,
  fileFilter,
  limits: { fileSize: 20 * 1024 * 1024 },   // 20 MB
});

```

```
1 MB = 1024 x 1024 bytes
20 MB = 20 x 1024 x 1024 = 20,971,520 bytes

```

---

## Route Mein Use Kaise Hota Hai

```js
import upload from "../config/multer.js";

router.post("/upload", upload.single("file"), (req, res) => {
  console.log(req.file);   // uploaded file ki details
  return res.json({ filename: req.file.filename });
});

```

### `.single("file")` ka matlab

Multer ke 4 methods hain — kitni files aayengi, uske hisaab se:

| **MethodKab use karo**      |                                              |
| --------------------------- | -------------------------------------------- |
| .single("fieldName")        | Sirf 1 file                                  |
| .array("fieldName", max)    | Multiple files, ek hi field naam se          |
| .fields([{name, maxCount}]) | Alag-alag field names (jaise resume + photo) |
| .none()                     | File nahi, sirf text fields                  |

**"file" string ka matlab:** Frontend jo naam bhejega, wahi yaha match hona chahiye:

```js
// Frontend
formData.append("file", selectedFile);   // "file"

// Backend
upload.single("file")                     // same "file"

```

Naam match na ho to `req.file` `undefined` reh jaayega — koi turant crash nahi, lekin `req.file.filename` access karte waqt crash hoga.

---

## Poori Journey — End to End Example

```
User "resume.pdf" (5MB) upload karta hai
   |
   v
Request /upload route pe pahunchti hai
   |
   v
Multer middleware chalta hai:
   |- fileFilter: mimetype === "application/pdf"? -> yes
   |- size check: 5MB < 20MB? -> yes
   |- destination(): "./temp" mein save karo
   |- filename(): "1758452891234-resume.pdf" naam do
   |
   v
File disk pe save: ./temp/1758452891234-resume.pdf
   |
   v
req.file = {
  filename: "1758452891234-resume.pdf",
  originalname: "resume.pdf",
  mimetype: "application/pdf",
  size: 5242880,
  path: "./temp/1758452891234-resume.pdf"
}
   |
   v
Route-handler chalta hai, req.file se aage use hota hai
(jaise PDF-agent isi path wali file process karta hai)

```

---

## Bugs Jo Pehli Baar Likhte Waqt Hue The

| **BugGalatSahiKyun** |                      |                          |                                                               |
| -------------------- | -------------------- | ------------------------ | ------------------------------------------------------------- |
| Folder crash         | fs.mkdir(dir, {...}) | fs.mkdirSync(dir, {...}) | Async mkdir ko callback chahiye hota hai, warna crash         |
| Garbage filename     | `${Date.now}`        | `${Date.now()}`          | Bina () ke function ka poora source-code string ban jaata hai |
| Property typo        | file.mimeType        | file.mimetype            | Multer ki property lowercase hai                              |
| Spelling typo        | "application/pfd"    | "application/pdf"        | Letters swap the                                              |
| Method typo          | .startWith(...)      | .startsWith(...)         | Beech mein "s" missing tha                                    |

---

## Zaroori Extra Concepts (jo abhi code mein nahi hain, lekin jaanne chahiye)

### 1. diskStorage vs memoryStorage

```js
multer.diskStorage({...})     // file seedha server ki hard-disk pe save hoti hai
multer.memoryStorage()        // file RAM mein rehti hai (req.file.buffer), disk pe save nahi hoti

```

**Kab kya use karo:**

- `diskStorage` — jab file ko baad mein kisi tool (jaise PDF-parser, image-resizer) ko **path se** dena ho, ya S3 pe upload se pehle temporarily rakhni ho.
- `memoryStorage` — jab file **chhoti** ho aur turant kisi cloud service (S3, Cloudinary) ko seedha bhej ke disk-save ki zarurat hi na ho — isse disk-cleanup ki tension bhi nahi rehti.

> Tumhare code mein `diskStorage` use ho raha hai — matlab file `./temp` folder mein reh jaati hai jab tak koi usko **khud delete** na kare. Ye agla point hai.

---

### 2. Temp files ka cleanup — ye abhi missing hai

Abhi ka code file ko save to karta hai, lekin **kabhi delete nahi karta**. Agar PDF-agent us file ko process karke result de bhi de, `./temp` folder mein file hamesha ke liye padi reh jaayegi.

**Problem:** Jitne zyada uploads honge, utna `./temp` folder bharta jaayega — ek din disk-space khatam ho sakta hai.

**Fix (jab bhi processing complete ho jaye):**

```js
import fs from "fs";

fs.unlink(req.file.path, (err) => {
  if (err) console.error("Temp file delete nahi hui:", err);
});

```

`fs.unlink` — file ko disk se delete karta hai. Isko route-handler ke **end mein** (chahe success ho ya error aaye) call karna best practice hai.

---

### 3. fileFilter ka mimetype sirf "label" hai, guarantee nahi

**Important security point:** `file.mimetype` browser/client khud bhejta hai — matlab ek attacker chahe to ek `.exe` file ka mimetype manually badal ke `"application/pdf"` bhej sakta hai. Multer ka `fileFilter` isko catch nahi karega, kyunki wo sirf client ne jo label bheja usi pe bharosa karta hai.

**Production-grade fix:** File ke andar ke actual bytes check karo ("magic number" check), jaise `file-type` naam ki npm library use karke:

```js
import { fileTypeFromFile } from "file-type";
const type = await fileTypeFromFile(req.file.path);
if (type?.mime !== "application/pdf") {
  // reject — mimetype label jhoothi thi
}

```

> Abhi ke liye tumhara code chalega, lekin agar kabhi ye system production mein jaaye jahan koi malicious user try kare, ye ek known gap hai jo yaad rakhna chahiye.

---

### 4. MulterError — jab size-limit cross ho jaaye

Jab file 20MB se badi ho, Multer khud ek `MulterError` throw karta hai — lekin agar tumne ise catch na kiya, Express ka default error-handler ek generic, ugly error dikhayega.

**Sahi tarika (error-handling middleware add karo):**

```js
app.use((err, req, res, next) => {
  if (err instanceof multer.MulterError) {
    if (err.code === "LIMIT_FILE_SIZE") {
      return res.status(400).json({ message: "File 20MB se badi hai" });
    }
    return res.status(400).json({ message: err.message });
  }
  next(err);
});

```

Ye middleware **route ke baad** (ya app ke end mein) lagana hota hai — Express ka convention hai ki error-handling middleware **4 arguments** leta hai (err, req, res, next), isse Express pehchanta hai ki ye khaas error-catcher hai.

---

## Interview Questions (Multer pe based)

**Q1. Multer kya hai aur ye kis problem ko solve karta hai?**

> Multer ek Express middleware hai jo `multipart/form-data` requests parse karta hai — ye woh format hai jo file uploads ke liye use hota hai. Express ka built-in `express.json()` sirf JSON body parse karta hai, files nahi — isliye Multer alag se chahiye hota hai.

**Q2. diskStorage aur memoryStorage mein kya farak hai?**

> `diskStorage` file ko server ki hard-disk pe save karta hai aur `req.file.path` deta hai. `memoryStorage` file ko RAM mein buffer ki tarah rakhta hai (`req.file.buffer`), disk pe kuch save nahi hota — chhoti files ke liye ya jab file seedha cloud (S3) pe forward karni ho, tab useful hai.

**Q3. fileFilter sirf mimetype check karta hai — kya ye 100% safe hai?**

> Nahi. `mimetype` client (browser) khud bhejta hai, isliye ise spoof kiya ja sakta hai — ek attacker `.exe` file ka mimetype badal ke `"application/pdf"` bhej sakta hai. Real security ke liye file ke actual bytes (magic number) check karne chahiye, jaise `file-type` library se.

**Q4. Agar koi 25MB ki file upload kare jab limit 20MB ho, to kya hoga?**

> Multer khud `MulterError` (code: LIMIT_FILE_SIZE) throw karega, request process nahi hoga. Isko handle karne ke liye ek dedicated error-handling middleware `(err, req, res, next) => {...}` lagana chahiye, warna generic/ugly error client ko dikhega.

**Q5. upload.single(), .array(), aur .fields() mein kab kaunsa use karoge?**

> `.single("field")` — ek hi file expect ho. `.array("field", max)` — multiple files, same field naam se (jaise ek saath 5 images). `.fields([...])` — alag-alag naam wale multiple fields, jaise ek form mein resume aur photo dono alag ho.

**Q6. Agar frontend formData.append("resume", file) bheje aur backend upload.single("file") likha ho, to kya hoga?**

> Field-naam match nahi karega, isliye Multer ko file milegi hi nahi. `req.file` undefined reh jaayega — turant crash nahi hoga, lekin jaise hi tum `req.file.filename` access karoge, "Cannot read properties of undefined" error aayega.

**Q7. destination aur filename functions ke andar cb(null, value) mein null kyun likhte hain?**

> Multer ka callback pattern Node.js ke standard error-first callback convention se aata hai — pehla argument hamesha "error hai ya nahi" batata hai. `null` ka matlab "koi error nahi hai", isliye dusra argument (actual value) use kiya jaayega. Agar error ho, pehla argument mein `new Error(...)` diya jaata (jaisa fileFilter mein reject karte waqt hota hai).

**Q8. Upload hone ke baad file ka kya hota hai — kya wo khud delete ho jaati hai?**

> Nahi, diskStorage use karne par file disk pe reh jaati hai jab tak use manually delete (fs.unlink) na kiya jaaye. Isliye processing complete hone ke baad temp files ko clean karna zaroori hai, warna disk-space dheere-dheere bharta jaata hai.

---

## Revision (30 second)

- Multer = file-upload middleware, khud "configure" kiya hai
- diskStorage -> kaha + kis naam se save hogi (vs memoryStorage -> RAM mein rehti hai)
- fileFilter -> allow/reject decide karta hai (mimetype check — spoofable hai, 100% safe nahi)
- limits.fileSize -> size cap; cross hone par MulterError aata hai, error-middleware se handle karo
- .single("field") -> field-naam frontend se match hona chahiye
- req.file -> uploaded file ki saari details yahan milti hain
- TODO: processing ke baad fs.unlink() se temp files delete karna mat bhoolna