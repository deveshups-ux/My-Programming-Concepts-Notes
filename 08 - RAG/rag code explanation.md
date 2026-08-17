# 📄 RAG (Retrieval Augmented Generation) — Full Code Explanation
### Tumhare Diye Gaye Real RAG Code Ka Line-by-Line Analysis

---

## 📄 Poora Code (Reference)

```javascript
import express from "express"
import dotenv from "dotenv"
import { ChatGoogleGenerativeAI } from "@langchain/google-genai"
import { ChatGroq } from "@langchain/groq"
import fs from "fs"
import { PDFParse } from "pdf-parse"
import { RecursiveCharacterTextSplitter } from "@langchain/textsplitters"
import { GoogleGenerativeAIEmbeddings } from "@langchain/google-genai"
import { TaskType } from "@google/generative-ai"
import { QdrantVectorStore } from "@langchain/qdrant"
import { HumanMessage, SystemMessage } from "@langchain/core/messages"

dotenv.config()
const app = express()
const port = 5000
app.use(express.json())

const llm = new ChatGroq({
    model: "llama-3.3-70b-versatile",
    temperature: 0.7,
    maxTokens: 100,
    maxRetries: 2
})

const embeddings = new GoogleGenerativeAIEmbeddings({
    model: "gemini-embedding-001", // 768 dimensions
    taskType: TaskType.RETRIEVAL_DOCUMENT,
    title: "Document title",
})

const vectorStore = await QdrantVectorStore.fromExistingCollection(embeddings, {
    url: process.env.QDRANT_URL,
    collectionName: "grocery-store",
})

const upload = async () => {
    const pdfPath = "./knowledge.pdf"
    const buffer = fs.readFileSync(pdfPath)
    const pdfResult = new PDFParse({ data: buffer })
    const result = await pdfResult.getText()
    const text = result.text
    const spilitter = new RecursiveCharacterTextSplitter({
        chunkSize: 1000,
        chunkOverlap: 200
    })
    const docs = await spilitter.createDocuments([text])
    await vectorStore.addDocuments(docs)
}

app.post("/ai", async (req, res) => {
    const { input } = req.body
    const docs = await vectorStore.similaritySearch(input, 5)
    const context = docs.map((d) => d.pageContent).join("/n")
    const response = await llm.invoke([
        new SystemMessage(`You are a RAG AI assistant.
STRICT RULES:
- Answer ONLY from context
- Do not use outside knowledge
- If answer not found say:
  "I don't know from uploaded PDF."
Context:
${context}`),
        new HumanMessage(input)
    ])
    console.log(response)
    return res.status(200).json({ ai: response.content })
})

app.get("/", (req, res) => {
    return res.json({ message: "hello from level4" })
})

app.listen(port, () => {
    console.log("server started")
})
```

---

## 1️⃣ `.env` File

```env
GOOGLE_API_KEY=add your
GROQ_API_KEY=add your
QDRANT_URL=add your
QDRANT_API_KEY=add your
```

| Key | Kis Liye |
|---|---|
| `GOOGLE_API_KEY` | Gemini embedding model call karne ke liye |
| `GROQ_API_KEY` | Groq LLM (Llama 3.3) ke liye — final answer generate karne ke liye |
| `QDRANT_URL` | Tumhare Qdrant vector database ka address (cloud ya local instance) |
| `QDRANT_API_KEY` | Qdrant database ko authenticate karne ke liye |

> ⚠️ **Note**: Code mein `QDRANT_API_KEY` import/env mein hai, lekin `QdrantVectorStore.fromExistingCollection()` call mein **pass nahi ki gayi** — sirf `url` aur `collectionName` diya hai. Agar tumhara Qdrant instance authentication maangta hai (jaise Qdrant Cloud), toh `apiKey: process.env.QDRANT_API_KEY` bhi add karna padega, warna connection fail ho sakta hai.

---

## 2️⃣ `package.json` Dependencies

| Package | Kaam |
|---|---|
| `@google/generative-ai` | Google ka raw SDK — yahan sirf `TaskType` enum ke liye use ho raha hai |
| `@langchain/core` | LangChain ka base package |
| `@langchain/google-genai` | Gemini LLM wrapper + Gemini Embeddings wrapper (dono isi package mein) |
| `@langchain/groq` | Groq LLM wrapper (final answer generation ke liye) |
| `@langchain/qdrant` | Qdrant vector database se LangChain ka connection |
| `@langchain/textsplitters` | Document ko chunks mein todne ke liye |
| `dotenv` | `.env` load karna |
| `express` | Backend server |
| `nodemon` | Dev auto-restart |
| `pdf-parse` | PDF file se raw text nikalna |

---

## 3️⃣ Imports — Kaunsa Kis Liye Aaya

```javascript
import { ChatGoogleGenerativeAI } from "@langchain/google-genai"
import { ChatGroq } from "@langchain/groq"
import fs from "fs"
import { PDFParse } from "pdf-parse"
import { RecursiveCharacterTextSplitter } from "@langchain/textsplitters"
import { GoogleGenerativeAIEmbeddings } from "@langchain/google-genai"
import { TaskType } from "@google/generative-ai"
import { QdrantVectorStore } from "@langchain/qdrant"
import { HumanMessage, SystemMessage } from "@langchain/core/messages"
```

| Import | Role |
|---|---|
| `ChatGoogleGenerativeAI` | Import hua hai but **use nahi hua** is code mein (Groq use ho raha hai final LLM ke liye) |
| `ChatGroq` | **Actual LLM** jo final answer generate karta hai |
| `fs` | Node.js ka built-in module — PDF file ko disk se **read** karne ke liye |
| `PDFParse` | PDF ke raw bytes se **text extract** karne ke liye |
| `RecursiveCharacterTextSplitter` | Bade text ko chhote **chunks** mein todne ke liye |
| `GoogleGenerativeAIEmbeddings` | Text ko **vectors** mein convert karne wala model |
| `TaskType` | Embedding model ko batata hai ki embedding **kis purpose** ke liye ban rahi hai (document store karne ke liye ya query search karne ke liye) |
| `QdrantVectorStore` | Qdrant database se connect/search karne ka wrapper |
| `HumanMessage, SystemMessage` | Structured message classes — role define karne ka **class-based tarika** (pehle wale code mein plain object `{role: "system", content: ""}` use hua tha, yahan class-based) |

---

## 4️⃣ LLM Setup (Final Answer Generator)

```javascript
const llm = new ChatGroq({
    model: "llama-3.3-70b-versatile",
    temperature: 0.7,
    maxTokens: 100,
    maxRetries: 2
})
```

**Kyun**: Ye wahi LLM hai jo user ke question ka **final answer** generate karega — lekin sirf uss context ke base par jo Vector DB se milega (RAG ka core idea).

> Same parameters jo LangGraph wale code mein the (`temperature`, `maxTokens`, `maxRetries`) — inka matlab already `01_LangChain_Notes.md` mein hai.

---

## 5️⃣ Embeddings Model Setup

```javascript
const embeddings = new GoogleGenerativeAIEmbeddings({
    model: "gemini-embedding-001", // 768 dimensions
    taskType: TaskType.RETRIEVAL_DOCUMENT,
    title: "Document title",
})
```

| Parameter | Matlab |
|---|---|
| `model: "gemini-embedding-001"` | Google ka embedding model jo text ko **768-dimension vector** mein convert karta hai |
| `taskType: TaskType.RETRIEVAL_DOCUMENT` | Batata hai ki ye embedding **document store karne** ke liye ban rahi hai (na ki query search ke liye) |
| `title: "Document title"` | Optional metadata — document ka title, embedding quality thoda improve kar sakta hai |

### 🔍 `TaskType` Kyun Important Hai (Deep Point)
Google ke embedding models mein **do alag modes** hote hain:
- `RETRIEVAL_DOCUMENT` → jab tum data ko **store** kar rahe ho (PDF ka content)
- `RETRIEVAL_QUERY` → jab tum **search query** ko embed kar rahe ho

> ⚠️ **Honest observation**: Is code mein `embeddings` object **sirf ek jagah** use ho raha hai — dono document upload (`upload()` function) aur query search (`vectorStore.similaritySearch()`) ke liye **same `taskType: RETRIEVAL_DOCUMENT`** use ho raha hai. Technically behtar practice ye hoti ki query search ke liye alag embeddings instance (`RETRIEVAL_QUERY` wala) use kiya jaye, taaki similarity search zyada accurate ho. Ye code **kaam kar jayega**, lekin ye ek **optimization gap** hai jo production-grade RAG mein fix kiya jaata hai.

---

## 6️⃣ Vector Store Connection

```javascript
const vectorStore = await QdrantVectorStore.fromExistingCollection(embeddings, {
    url: process.env.QDRANT_URL,
    collectionName: "grocery-store",
})
```

**Kya kar raha hai**: Qdrant (ek Vector Database) ke **existing collection** (jaise ek table/bucket) se connect ho raha hai.

| Part | Matlab |
|---|---|
| `embeddings` | Konsa embedding model use karna hai, taaki store/search dono consistent rahein |
| `url` | Qdrant server ka address |
| `collectionName: "grocery-store"` | Collection (jaise database table) ka naam — is naam ki collection Qdrant mein already exist honi chahiye |

> **Note**: `fromExistingCollection` ka matlab hai collection **pehle se bani honi chahiye** Qdrant mein. Agar collection exist nahi karti, ye function fail ho sakta hai — naya collection banane ke liye `fromDocuments()` ya `fromTexts()` jaisa method use hota hai (is code mein use nahi hua).

---

## 7️⃣ Upload Function — PDF Se Vector DB Tak (Indexing Pipeline)

```javascript
const upload = async () => {
    const pdfPath = "./knowledge.pdf"
    const buffer = fs.readFileSync(pdfPath)
    const pdfResult = new PDFParse({ data: buffer })
    const result = await pdfResult.getText()
    const text = result.text
    const spilitter = new RecursiveCharacterTextSplitter({
        chunkSize: 1000,
        chunkOverlap: 200
    })
    const docs = await spilitter.createDocuments([text])
    await vectorStore.addDocuments(docs)
}
```

Ye function **RAG ka "Indexing" phase** hai — matlab data ko taiyaar karke Vector DB mein daalna.

### Step-by-Step

| Step | Line | Kya Ho Raha Hai |
|---|---|---|
| 1 | `fs.readFileSync(pdfPath)` | PDF file ko disk se **raw bytes (buffer)** mein read kiya |
| 2 | `new PDFParse({ data: buffer })` | PDF parser object banaya |
| 3 | `pdfResult.getText()` | PDF se **plain text** extract kiya |
| 4 | `RecursiveCharacterTextSplitter({ chunkSize: 1000, chunkOverlap: 200 })` | Text splitter banaya — bada text chhote-chhote **1000-character chunks** mein todega |
| 5 | `chunkOverlap: 200` | Har chunk ke **200 characters previous chunk se overlap** karenge (taaki context beech mein na tute) |
| 6 | `spilitter.createDocuments([text])` | Text ko actually chunks mein todkar `Document` objects ki list banayi |
| 7 | `vectorStore.addDocuments(docs)` | Har chunk ko **embedding model se vector mein convert** karke Qdrant mein store kiya |

### 🎯 `chunkOverlap` Kyun Zaroori Hai (Important Concept)
```
Chunk 1: "...the recipe requires 200g flour and 3 eggs..."
Chunk 2 (bina overlap): "...mix well before baking at 180°C..."
```
Agar overlap na ho, toh important context (jaise "kis cheez ko 180°C pe bake karna hai") **do chunks ke beech kat sakta hai**. `chunkOverlap: 200` isliye rakha hai taaki har chunk ke end ka thoda hissa agle chunk mein bhi repeat ho — continuity bani rahe.

### ⚠️ Important Observation
Ye `upload()` function **kahin bhi call nahi ho raha** is code mein — na kisi route se, na startup pe. Matlab abhi ye **dead code** hai jab tak koi explicitly isko trigger na kare (jaise ek `/upload` POST route banake, ya server start hote hi ek baar call karke).

---

## 8️⃣ Main RAG Route — Query Se Answer Tak (Retrieval Phase)

```javascript
app.post("/ai", async (req, res) => {
    const { input } = req.body
    const docs = await vectorStore.similaritySearch(input, 5)
    const context = docs.map((d) => d.pageContent).join("/n")
    const response = await llm.invoke([
        new SystemMessage(`You are a RAG AI assistant.
STRICT RULES:
- Answer ONLY from context
- Do not use outside knowledge
- If answer not found say:
  "I don't know from uploaded PDF."
Context:
${context}`),
        new HumanMessage(input)
    ])
    return res.status(200).json({ ai: response.content })
})
```

### Step-by-Step

| Step | Line | Kya Ho Raha Hai |
|---|---|---|
| 1 | `const { input } = req.body` | User ka question liya |
| 2 | `vectorStore.similaritySearch(input, 5)` | User ke question ko embed karke, Qdrant mein **top 5 semantically similar chunks** dhoondhe |
| 3 | `docs.map((d) => d.pageContent).join("/n")` | Un 5 chunks ka text nikal ke ek **single context string** bana di |
| 4 | `new SystemMessage(...)` | LLM ko **strict instruction** di gayi — sirf context se answer do, bahar ka knowledge use mat karo |
| 5 | `new HumanMessage(input)` | User ka original question bhi bheja |
| 6 | `llm.invoke([...])` | LLM ne context + question dekh ke answer generate kiya |
| 7 | `response.content` | Final answer client ko return kiya |

### 🐛 Chhoti Si Bug (Important — Batana Zaroori Hai)
```javascript
docs.map((d) => d.pageContent).join("/n")
```
Yahan `"/n"` likha hai — ye **string** hai, newline character nahi. Actual newline ke liye `"\n"` hona chahiye (backslash, forward-slash nahi). Abhi ye code **kaam toh karega**, lekin chunks ke beech literal `/n` text insert ho jayega instead of ek proper line break. Chhota cosmetic bug hai, functionality nahi todega, lekin context thoda "messy" dikh sakta hai LLM ko.

### 🔒 System Prompt Ka "Strict RAG" Pattern (Bahut Important Concept)
```
STRICT RULES:
- Answer ONLY from context
- Do not use outside knowledge
- If answer not found say: "I don't know from uploaded PDF."
```
Ye prompt engineering ka ek **critical RAG pattern** hai — isko **"grounding"** kehte hain. Iske bina, LLM apni training knowledge se bhi answer de sakta hai (hallucinate kar sakta hai), jo RAG ka **poora purpose hi khatam** kar deta hai. Isliye explicitly bola gaya hai:
- Sirf diye gaye context se answer do
- Agar context mein jawab nahi hai, "pata nahi" bolo (galat jawab mat do)

---

## 9️⃣ Poora RAG Flow — Do Phases Mein Samjho

### 📥 PHASE 1: Indexing (upload function — one-time setup)
```
PDF File
   ↓ (fs.readFileSync)
Raw Buffer
   ↓ (PDFParse.getText())
Plain Text
   ↓ (RecursiveCharacterTextSplitter)
Chunks (1000 chars each, 200 overlap)
   ↓ (embeddings model)
Vectors
   ↓ (vectorStore.addDocuments)
Qdrant Vector Database (stored)
```

### 📤 PHASE 2: Retrieval + Generation (har /ai request pe)
```
User Question
   ↓ (embeddings model — query ko bhi vector banaya)
Query Vector
   ↓ (vectorStore.similaritySearch)
Top 5 Similar Chunks
   ↓ (join into context string)
Context Text
   ↓ (SystemMessage + HumanMessage)
LLM (Groq)
   ↓
Final Answer (grounded in PDF content)
```

---

## 🔟 Real Scenario Walkthrough

**Suppose PDF mein likha hai**: *"Grocery store 9 AM se 9 PM tak khula rehta hai. Sunday band rehta hai."*

**User pucho**: *"Store kab khulta hai?"*

```
1. Request /ai pe aata hai
2. vectorStore.similaritySearch("Store kab khulta hai?", 5)
   → Query embed hoti hai, Qdrant mein similar chunks dhoondhe jaate hain
3. Relevant chunk milta hai: "Grocery store 9 AM se 9 PM tak khula rehta hai..."
4. context variable mein ye text aa jaata hai
5. LLM ko System Prompt (strict rules) + context + question diya jaata hai
6. LLM answer deta hai: "Store 9 AM se 9 PM tak khula rehta hai"
```

**User pucho (kuch alag jo PDF mein nahi hai)**: *"Store ka phone number kya hai?"*
```
1. similaritySearch chalta hai, lekin PDF mein phone number ka koi mention nahi
2. Retrieved chunks mein relevant info nahi milti
3. System prompt ke rules ke wajah se LLM bolta hai:
   "I don't know from uploaded PDF."
   (Hallucinate nahi karta, galat number nahi banata)
```

---

## 1️⃣1️⃣ Is Code Mein Kya Missing/Improvable Hai (Sabhi Honest Points)

| Issue | Kya Hai | Impact |
|---|---|---|
| `upload()` kabhi call nahi hota | Koi route/trigger nahi hai isko chalane ka | PDF kabhi upload/index nahi hoga jab tak manually call na karo |
| `"/n"` vs `"\n"` | String bug | Chunks ke beech literal text aa jaayega, functional nahi todega |
| Same `taskType` for query aur document | `RETRIEVAL_DOCUMENT` dono jagah use ho raha | Similarity search thoda kam accurate ho sakta hai (`RETRIEVAL_QUERY` behtar hota query ke liye) |
| `QDRANT_API_KEY` set hai but pass nahi hui | `.env` mein hai, code mein use nahi | Agar Qdrant Cloud auth maangta hai, connection fail ho sakta hai |
| Fixed `k=5` results | Hardcoded `5` similarity results | Kabhi zyada, kabhi kam relevant chunks chahiye ho sakte hain — dynamic karna behtar hota |

> Ye sab **chhote improvements** hain — code ka **core RAG concept bilkul sahi** implement hua hai. Ye issues sirf "production-polish" level ke hain.

---

## 1️⃣2️⃣ Ye Code RAG Ke Kaunse Theory Concepts Practically Dikhata Hai

| PDF Theory Concept | Is Code Mein Kahan |
|---|---|
| Chunking | `RecursiveCharacterTextSplitter` |
| Embeddings | `GoogleGenerativeAIEmbeddings` |
| Vector Database | `QdrantVectorStore` |
| Similarity Search | `vectorStore.similaritySearch(input, 5)` |
| LLM + Context = Answer | `llm.invoke([SystemMessage(context), HumanMessage(input)])` |

**Ab tumhara pura RAG theory-to-practice mapping complete ho gaya hai** — jo pehle sirf slides mein tha, ab real code mein exactly dikh raha hai.

---

## 📊 Updated Coverage (RAG Specific)
```
RAG Theory:      ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓░░░░  ~80%
RAG Practical:   ▓▓▓▓▓▓▓▓▓▓░░░░░░░░░░  ~50% (ek real example dekha/samjha, khud nahi banaya)
```
> Estimate hai — tumne ek **complete working RAG code dekha aur samjha** hai (indexing + retrieval dono phases), lekin khud se scratch se likha/debug/deploy nahi kiya hai abhi. Isliye "practical" 50% — pura hands-on experience nahi, lekin sirf-theory se kaafi aage.

---

## 🎯 Ab Tum Kya Bana Sakte Ho (RAG-Specific)
- ✅ Samajh sakte ho ki PDF-chatbot kaise kaam karta hai end-to-end
- ✅ Pehchaan sakte ho RAG code mein bugs/improvements (jaisa upar dikhaya)
- ⚠️ Khud se scratch se RAG likhना — abhi tumne sirf ek example dekha hai, khud independently likhne ka practice baaki hai