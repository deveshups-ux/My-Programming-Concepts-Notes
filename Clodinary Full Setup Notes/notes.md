# Cloudinary Upload Notes (Next.js / Node.js)

## 1. Install
```bash
npm install cloudinary
```

## 2. Env Variables (.env.local)
```
CLOUDINARY_CLOUD_NAME=your_cloud_name
CLOUDINARY_API_KEY=your_api_key
CLOUDINARY_API_SECRET=your_api_secret
```
> Ye 3 values Cloudinary Dashboard se milti hain. `api_secret` kabhi frontend/client pe expose mat karna — sirf server-side.

## 3. Config Setup (lib/cloudinary.ts)
```ts
import { v2 as cloudinary } from "cloudinary";

cloudinary.config({
  cloud_name: process.env.CLOUDINARY_CLOUD_NAME,
  api_key: process.env.CLOUDINARY_API_KEY,
  api_secret: process.env.CLOUDINARY_API_SECRET,
});
```

## 4. Upload Function (Full Utility)
```ts
const uploadOnCloudinary = async (
  file: Blob | null
): Promise<string | null> => {
  if (!file) return null;

  try {
    const arrayBuffer = await file.arrayBuffer();
    const buffer = Buffer.from(arrayBuffer);

    return new Promise((resolve, reject) => {
      const uploadStream = cloudinary.uploader.upload_stream(
        { resource_type: "auto" },
        (error, result) => {
          if (error) reject(error);
          else resolve(result?.secure_url ?? null);
        }
      );
      uploadStream.end(buffer);
    });
  } catch (error) {
    console.log(error);
    return null;
  }
};

export default uploadOnCloudinary;
```

## 5. Step-by-Step Flow (Yaad rakhne ke liye)
1. `file` (Blob) aata hai → check null
2. `file.arrayBuffer()` → binary data nikala
3. `Buffer.from(arrayBuffer)` → Node.js compatible Buffer banaya
4. `upload_stream()` → Cloudinary ko stream diya (callback-based hai)
5. Callback ke andar `resolve()`/`reject()` call karke Promise complete kiya
6. `.end(buffer)` → actual upload trigger hota hai
7. Success → `secure_url` milta hai (HTTPS link, DB mein save karo)
8. Fail → `null` return hota hai

## 6. Kaha se Call Karna Hai (API Route Example)
```ts
import uploadOnCloudinary from "@/lib/cloudinary";

const formData = await req.formData();
const file = formData.get("file") as Blob;

const url = await uploadOnCloudinary(file);

if (!url) {
  return Response.json({ error: "Upload failed" }, { status: 500 });
}

// url ko database mein save karo
```

## 7. Common Mistakes / Reminders
- [ ] `.env.local` mein keys daalna mat bhoolna
- [ ] `resource_type: "auto"` rakho agar image/video/pdf sab allow karne hain
- [ ] `api_secret` kabhi client-side code mein use mat karo
- [ ] Bahar se call karte waqt `try-catch` lagao (kyunki `reject()` wali error bahar propagate hoti hai)
- [ ] Bade files ke liye stream hi use karo, direct base64/buffer upload avoid karo

## 8. Quick Concept Recap
| Concept | Iska Kaam |
|---|---|
| Callback | Async kaam complete hone pe wapas call hota hai |
| Stream | Data ko chunks mein bhejta hai (memory-efficient) |
| Promise | Async kaam ka "future result" — resolve/reject |
| async/await | Promise ko synchronous jaisa likhne ka tareeka |
