# AWS S3 — Complete Notes

---

## 1. Quick Reference (TL;DR)

| Term | Matlab |
|---|---|
| **S3** | Amazon ki cloud file storage service |
| **Bucket** | Ek "folder" jaisa container jisme files rakhi jaati hain |
| **Key** | File ka naam/path bucket ke andar |
| **Region** | Kis geographic location ka server use ho raha hai |
| **Buffer** | File ka raw binary data (0s aur 1s), jo upload se pehle memory mein hold hota hai |
| **Presigned URL** | Ek temporary, time-limited, cryptographically signed link jo private file ko access karne deta hai |
| **PutObjectCommand** | Instruction — "is bucket mein ye file daal do" |
| **GetObjectCommand** | Instruction — "is bucket se ye file nikaal do" |

---

## 2. S3 Kya Hai (What & Why)

### Kya hai
S3 (Simple Storage Service) Amazon (AWS) ki ek service hai jiska sirf ek kaam hai — **files store karna, cloud pe**. Ye insaan ke liye nahi, apps/programs ke liye bana hai.

### Kyu aaya (why it exists)
Pehle apps apni files apne hi server ke disk pe store karte the. Isme teen badi dikkatein thi:

1. **Storage limited hoti hai** — server ka disk space fix hota hai, khatam ho sakta hai
2. **Cloud servers pe disk temporary hoti hai** — agar server restart/redeploy ho (jaise Render, Heroku pe), to us disk ki saari files **gayab ho jaati hain**
3. **Scale karna mushkil hai** — agar hazaaron users ek saath file access karein, server overload ho sakta hai

**Solution:** Ek alag, dedicated service banayi jaye jiska kaam hi sirf files store karna ho, reliably aur bina limit ke. Yahi S3 hai.

### Kab/Kaha use karte hain
Jab bhi app mein **koi bhi file** (image, PDF, video, document) store karni ho jo server restart ke baad bhi zinda rahe — jaise profile pictures, uploaded documents, generated reports, wagera.

---

## 3. Core Concepts / Terminology

### Bucket
Ek container/folder jaisi cheez S3 mein, jahan aap apni files rakhte ho. Har bucket ka **globally unique naam** hota hai (poori duniya mein koi doosra bucket same naam ka nahi ho sakta).

**Example:** `my-kaido-app-files`

### Key
Bucket ke andar file ka naam/path. Ye seedha ek string hoti hai, folders jaisa dikh sakta hai lekin S3 mein actually "folders" nahi hote — bas naming convention hoti hai.

**Example:** `users/123/avatar.png` (ye ek hi "Key" hai, chahe ye folder jaisa dikhta ho)

### Region
AWS ke servers duniya bhar mein alag-alag jagah (data centers) hote hain — jaise Mumbai (`ap-south-1`), Virginia (`us-east-1`). Aapko bucket banate waqt ek region choose karna padta hai. Jitna closer region user ke, utna fast access.

### Credentials (Access Key + Secret Key)
Ye aapki AWS account ki **login details** hain — jaise username-password.
- `accessKeyId` — username jaisa (kaun request bhej raha hai)
- `secretAccessKey` — password jaisa (proof ki ye genuine hai)

**Kyu zaroori:** Bina in dono ke, AWS kisi ko bhi kuch karne nahi deta — security ke liye.

---

## 4. Setup — S3 Client Banana

```js
import { S3Client } from "@aws-sdk/client-s3";

export const s3 = new S3Client({
  region: process.env.AWS_REGION,
  credentials: {
    accessKeyId: process.env.AWS_ACCESS_KEY_ID,
    secretAccessKey: process.env.AWS_SECRET_KEY,
  },
});
```

### Kya ho raha hai
- `S3Client` — AWS SDK ka wo cheez jo S3 se "baat karne" ka connection banati hai
- `new S3Client({...})` — ek connection object banaya, jisse poore app mein reuse karenge
- `region` / `credentials` — `.env` file se aa rahe hain, taaki secret values code mein hardcode na ho (security)

### Kaha use hoga
Ye ek baar banega, aur jahan bhi S3 se kaam karna ho (upload, download), wahan yahi `s3` object import karke use hoga.

### `.env` mein kya-kya chahiye
```
AWS_REGION=ap-south-1
AWS_ACCESS_KEY_ID=your_access_key
AWS_SECRET_KEY=your_secret_key
AWS_BUCKET_NAME=your_bucket_name
```

---

## 5. Upload Function

```js
import { PutObjectCommand } from "@aws-sdk/client-s3";
import { s3 } from "../config/s3.js";

export const uploadToS3 = async (filename, buffer, contentType) => {
  await s3.send(
    new PutObjectCommand({
      Bucket: process.env.AWS_BUCKET_NAME,
      Body: buffer,
      Key: filename,
      ContentType: contentType,
    }),
  );
  return filename;
};
```

### Buffer kya hai
Har file (image, PDF, video) asal mein **raw binary data** hoti hai — 0 aur 1 ka lamba sequence. `Buffer` Node.js ka data-type hai jo is raw data ko memory mein hold karta hai taaki use process/upload kar sako.

**Example:** Jab koi image upload hoti hai (jaise Multer middleware se), wo `req.file.buffer` mein milti hai — kuch aisi dikhti hai: `<Buffer ff d8 ff e0 00 10 ...>` — ye numbers image ka actual data hain.

### Line-by-line
- `filename` — S3 mein file kis naam se save hogi
- `buffer` — actual raw file data
- `contentType` — file kis type ki hai (`image/png`, `application/pdf`) — isse pata chalta hai browser ko file kaise render karni hai
- `PutObjectCommand({...})` — instruction packet: "is bucket mein, is naam se, ye data daal do"
- `s3.send(...)` — actually request bhejta hai AWS ko, execute karwata hai
- `return filename` — taaki caller ko pata rahe file kis naam se save hui (database mein save karne ke liye)

### Kab/Kaise use hoga
Jab bhi koi file upload honi ho — jaise user profile picture bhejta hai, ya koi document attach karta hai:
```js
const filename = `users/${userId}/avatar-${Date.now()}.png`;
await uploadToS3(filename, req.file.buffer, req.file.mimetype);
```

---

## 6. Presigned URL — Kya, Kyu, Kaise

### Problem jo ye solve karta hai
S3 mein files **by default private** hoti hain — koi bhi seedha URL khol ke file access nahi kar sakta (security ke liye achha hai). Lekin legitimate user ko bhi to file dikhani hai — kaise?

### Solution — Presigned URL
Ek **temporary, time-limited link** banate hain jo thodi der ke liye (jaise 10 min) us private file ko access karne deta hai, phir apne aap expire ho jaata hai.

### "Presigned" naam kyu
- **"Pre"** = pehle se (advance mein)
- **"Signed"** = cryptographically verify/prove kiya hua

Matlab: **URL banate waqt hi** (advance mein) ek proof/signature usme embed kar diya jaata hai — jisse use karte waqt baar-baar authenticate karne ki zaroorat nahi padti.

**Analogy:** Concert ticket jaisa — counter pe ID dikha ke ticket lete ho (signing), phir gate pe sirf ticket dikhana hota hai (bina dobara ID dikhaye) — jab tak ticket expire na ho jaaye.

### Code
```js
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";
import { s3 } from "../config/s3.js";
import { GetObjectCommand } from "@aws-sdk/client-s3";

export const getFromS3 = async (filename, expiresIn = 600) => {
  return await getSignedUrl(
    s3,
    new GetObjectCommand({
      Bucket: process.env.AWS_BUCKET_NAME,
      Key: filename,
    }),
    { expiresIn },
  );
};
```

### Line-by-line
- `getSignedUrl` — special helper jo temporary URLs banata hai (alag package: `s3-request-presigner`)
- `GetObjectCommand({...})` — request template: "is bucket se, ye file nikaalo"
- `expiresIn = 600` — URL kitni der (seconds mein) valid rahega — default 10 minutes
- Return value — ek URL string jismein end mein signature/expiry embedded hota hai:
  ```
  https://mybucket.s3.amazonaws.com/photo.png?X-Amz-Signature=abc123...&X-Amz-Expires=600
  ```

### Kab/Kaise use hoga
Jab bhi frontend ko koi private file dikhani ho:
```js
const url = await getFromS3("users/123/avatar.png", 300); // 5 min valid
res.json({ url });
// Frontend: <img src={url} />
```
5 minute baad wahi URL "Access Denied" dega — naya URL banana padega.

---

## 7. End-to-End Real Usage Example

```js
// Route: POST /upload-avatar
export const uploadAvatar = async (req, res) => {
  const userId = req.user.id;
  const filename = `users/${userId}/avatar-${Date.now()}.png`;

  await uploadToS3(filename, req.file.buffer, req.file.mimetype);

  // Database mein filename save karo (URL nahi, kyunki URL expire ho jaata hai)
  await User.findByIdAndUpdate(userId, { avatarKey: filename });

  const url = await getFromS3(filename);
  res.json({ url });
};
```

**Zaroori baat:** Database mein hamesha `filename` (Key) save karo, **signed URL nahi** — kyunki URL expire ho jaata hai, lekin filename permanent hai. Jab bhi file dikhani ho, us filename se naya signed URL generate kar lo.

---

## 8. Visual Flow

```
[Upload Time]
User → Backend (file as Buffer) → uploadToS3() → S3 Bucket (private)
                                                      ↓
                                          filename database mein save

[Access Time]
Frontend request karta hai → Backend → getFromS3(filename)
                                             ↓
                              Temporary Signed URL (10 min valid)
                                             ↓
                              Frontend ko URL milta hai → <img src={url}>

[After Expiry]
Same URL access karne pe → "Access Denied"
→ Dobara getFromS3() call karna padega naya URL ke liye
```

---

## 9. Security Best Practices

1. **`.env` file ko kabhi commit mat karo** — `.gitignore` mein add karo, warna credentials GitHub pe public ho jaayenge
2. **Least Privilege** — AWS IAM user banate waqt, poore account ka access mat do; sirf specific bucket ke liye limited permissions (upload/read) do
3. **Bucket "Block Public Access" ON rakho** — jab tak specifically public files nahi chahiye
4. **Credentials leak ho jaayein to turant rotate karo** — AWS console se purani key delete karke nayi banao
5. **Signed URL ka expiry time chhota rakho** — jitni der chahiye utni hi do, zyada mat do (security risk kam hota hai)

---

## 10. Error Handling

S3 calls fail bhi ho sakte hain — network issue, galat credentials, bucket na milna, wagera. Common errors:

| Error | Matlab |
|---|---|
| `InvalidAccessKeyId` | Access key galat hai |
| `SignatureDoesNotMatch` | Secret key galat hai |
| `NoSuchBucket` | Bucket naam galat hai ya exist nahi karta |
| `AccessDenied` | Permissions sahi nahi hain (IAM policy check karo) |

**Best practice:** Har S3 call ko try/catch mein wrap karo:
```js
try {
  await uploadToS3(filename, buffer, contentType);
} catch (error) {
  console.error("S3 upload failed:", error);
  // graceful fallback / user ko error message
}
```

---

## 11. Cost Understanding

S3 ka billing 3 cheezon pe depend karta hai:

1. **Storage** — kitna data total store kiya hai (per GB/month)
2. **Requests** — kitni baar upload/download/list call hui (per 1000 requests)
3. **Data Transfer (Bandwidth)** — kitna data bahar (internet pe) gaya — ye usually sabse mehenga part hota hai

**Tip:** Agar app scale kare aur bahut saari files baar-baar download/view ho rahi hon, bandwidth cost badh sakta hai. Isiliye CDN (CloudFront) use karna better hota hai bade scale pe.

---

## 12. Common Bugs Jo Humne Face Kiye (Real Lessons)

### Bug 1: Export/Import naam mismatch
```js
// s3.js mein
export const client = new S3Client({...});

// dusri file mein
import { s3 } from "../config/s3.js";  // undefined milega!
```
**Lesson:** Export aur import ka naam **exactly** match hona chahiye — JavaScript case-sensitive hai.

### Bug 2: `expireIn` vs `expiresIn` typo
```js
{ expireIn }  // galat — AWS SDK isko silently ignore kar dega
{ expiresIn } // sahi
```
**Lesson:** Chhota typo crash nahi karta, lekin feature silently kaam nahi karta — default value use ho jaati hai.

### Bug 3: File extension consistency
```js
import { s3 } from "../config/s3";     // ek jagah
import { s3 } from "../config/s3.js";  // dusri jagah
```
**Lesson:** Strict ES modules mein `.js` extension zaroori hota hai — consistency rakho poore project mein.

---

## 13. Abhi Jo Missing Hai (Future Improvements)

| Feature | Kyu chahiye |
|---|---|
| **Delete function** (`DeleteObjectCommand`) | Files remove karne ke liye — abhi koi tarika nahi hai |
| **Unique filename generation** | Abhi agar do log same naam ("photo.png") se upload karein, ek doosre ko overwrite kar dega |
| **File size/type validation** | Abhi koi limit nahi — koi bhi size/type ki file upload ho sakti hai |
| **Direct browser-to-S3 upload** (presigned PUT) | Bade files ke liye backend bandwidth bachane ke liye |
| **List objects** (`ListObjectsV2Command`) | Kisi folder/user ki saari files dekhne ke liye |
| **Multipart upload** | Bahut badi files (jaise videos) ke liye reliable upload |

**Unique filename ka quick fix:**
```js
import { randomUUID } from "crypto";
const uniqueFilename = `${randomUUID()}-${originalFilename}`;
```

---

## 14. S3 vs Cloudinary — Comparison

| | **S3** | **Cloudinary** |
|---|---|---|
| Kya hai | Generic storage (kuch bhi) | Image/video-specific, storage + processing |
| Setup | Thoda complex (client, presigned URLs) | Bahut simple (1-2 line code) |
| Image resize/crop | Khud code likhna padta hai | Built-in URL params se |
| CDN | Alag se CloudFront setup | Built-in automatic |
| Best for | Generic files, bade scale, AWS ecosystem | Images/videos, jaldi setup, kam code |

**Bottom line:** Sirf images/videos ke liye jaldi setup chahiye → Cloudinary. Generic files, bada scale, ya AWS ecosystem mein already ho → S3.

**Alternative:** Cloudflare R2 — S3-compatible, lekin bandwidth (download) FREE hota hai — bade scale pe bahut sasta.

---

## 15. FAQ / Common Confusions

**Q: Buffer kya hota hai?**
A: File ka raw binary data jo memory mein temporarily hold hota hai upload se pehle.

**Q: "Presigned" ka matlab kya hai?**
A: URL banate waqt hi (advance/"pre") ek cryptographic proof ("sign") usme embed kar diya jaata hai, taaki baad mein baar-baar authenticate na karna pade.

**Q: Database mein signed URL save karna chahiye ya filename?**
A: Hamesha filename (Key) save karo — URL expire ho jaata hai, filename permanent rehta hai.

**Q: Agar do users same filename se upload karein to kya hoga?**
A: Ek doosre ko overwrite kar dega (data loss!) — isliye unique filename generation zaroori hai (UUID add karke).

**Q: Bucket public karna chahiye ya private?**
A: Default private rakho, presigned URLs use karo access ke liye — jab tak koi specific reason na ho public rakhne ka.

---

## 16. Glossary

| Term | Definition |
|---|---|
| **Bucket** | S3 ka storage container, jaise ek top-level folder |
| **Key** | File ka unique naam/path bucket ke andar |
| **Region** | AWS data center ki geographic location |
| **Buffer** | Raw binary file data, memory mein hold hua |
| **ContentType** | File ka MIME type (`image/png`, `application/pdf`) |
| **Presigned URL** | Temporary, signed, time-limited access link |
| **IAM** | AWS ka permission/access management system |
| **CDN** | Content Delivery Network — files ko duniya bhar mein fast deliver karne ka system |

---

## 17. Useful Links

- AWS S3 Official Docs: https://docs.aws.amazon.com/s3/
- AWS SDK v3 for JavaScript: https://www.npmjs.com/package/@aws-sdk/client-s3
- Presigned URL Guide: https://docs.aws.amazon.com/AmazonS3/latest/userguide/ShareObjectPreSignedURL.html