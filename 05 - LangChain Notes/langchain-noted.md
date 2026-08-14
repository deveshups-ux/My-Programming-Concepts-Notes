# 🦜 LangChain — Complete Notes
### Covered Topics + Pending Roadmap

---

## 📊 Coverage Status
```
LangChain Progress:  ▓▓▓░░░░░░░░░░░░░░░░░  ~15-20% (rough estimate, honest)
```
> Ye ek rough estimate hai based on kitne core LangChain features tumne touch kiye hain — exact measured number nahi hai.

---

# ✅ PART 1 — Jo Tumne Ab Tak Cover Kiya

## 1. What is LangChain?

LangChain ek **framework** hai jo LLM applications banane ko simplify karta hai.

```
LangChain connects: LLM + Memory + Tools + Vector DB + APIs
```

**Kya banata hai**: AI chatbots, RAG systems, AI agents, tool-calling apps, multi-step workflows.

---

## 2. LLM Integration / Calling

Do alag LLMs ko LangChain ke through call karna seekha:

```javascript
// Gemini
import { ChatGoogleGenerativeAI } from "@langchain/google-genai"

// Groq (Llama 3.3)
import { ChatGroq } from "@langchain/groq"

const llm = new ChatGroq({
    model: "llama-3.3-70b-versatile",
    temperature: 0.7,
    maxTokens: 100,
    maxRetries: 2
})
```

**Key Insight**: LangChain **same interface** (`.invoke()`) se alag-alag companies ke LLMs call karne deta hai — model switch karna aasan hai, code structure same rehta hai.

---

## 3. Role System (System / Human Messages)

```javascript
await llm.invoke([
    { role: "system", content: "You are Jarvis AI assistant..." },
    { role: "human", content: userInput }
])
```

| Role | Kaam |
|---|---|
| `system` | AI ka behaviour/personality/rules define karta hai |
| `human` | User ka actual message |

> Raw SDK (bina LangChain) mein ye roles manually format karne padte hain; LangChain isko simplify karta hai.

---

## 4. LLM Parameters

| Parameter | Matlab |
|---|---|
| `temperature` | 0 = factual/strict, 1 = creative/random |
| `maxTokens` | Response ki max length |
| `maxRetries` | Fail hone par kitni baar retry |

---

## 5. `.env` Auto API Key Pickup

LangChain ke wrappers (`ChatGoogleGenerativeAI`, `ChatGroq`) apne corresponding env variable **khud dhoondh lete hain** — `apiKey` manually pass karne ki zaroorat nahi:

```env
GOOGLE_API_KEY="..."
GROQ_API_KEY="..."
```

> Raw SDK (`@google/genai`) mein ye automatic nahi hota, wahan manually `apiKey: process.env.GEMINI_API_KEY` dena padta hai.

---

## 6. `.bindTools()` — Tool Binding

```javascript
const llm = new ChatGroq({ ...config }).bindTools(tools)
```

**Kya karta hai**: LLM ko batata hai "tumhare paas ye tools available hain, zaroorat pade toh use kar sakte ho". Bina isske LLM ko tools ka pata hi nahi chalega.

> Ye concept LangChain ka hai, lekin humne isko **LangGraph ke andar** use kiya (agent workflow banate waqt).

---

# ❌ PART 2 — Jo Abhi Baaki Hai (Real LangChain Roadmap)

## 1. Prompt Templates
Dynamic, reusable prompts banane ka tarika — variables ke saath.
```javascript
// Aisa kuch hota hai (abhi nahi seekha)
const template = ChatPromptTemplate.fromTemplate("Translate {text} to {language}")
```
**Kyun important**: Hardcoded strings ki jagah reusable templates — production apps mein zaroori.

---

## 2. LCEL (LangChain Expression Language) / Chains
```
prompt | llm | parser
```
Ye **pipe syntax** LangChain ka core naming reason hai — multiple steps ko chain (zanjeer) ki tarah jodna.

**Abhi tum kya kar rahe ho instead**: Direct `.invoke()` call — chaining nahi ho rahi.

---

## 3. Output Parsers / Structured Output
LLM ka raw text response ko **structured JSON/object** mein convert karna.
```javascript
// withStructuredOutput() jaisa kuch (abhi nahi seekha)
```
**Kyun important**: Agar tumhe LLM se guaranteed format mein data chahiye (jaise `{name: "", age: 0}`), plain text kaafi nahi hota.

---

## 4. Custom Tools
Abhi sirf **pre-built** `TavilySearch` tool use kiya hai. Khud ka tool banana:
```javascript
// tool() function se apna custom tool (abhi nahi seekha)
```
**Example use-case**: Apna khud ka "database query tool" ya "calculator tool" banana.

---

## 5. Document Loaders
PDF, CSV, Word, websites se data load karna (RAG ka pehla step).

---

## 6. Text Splitters (Chunking)
Bade documents ko chhote chunks mein todna, taaki embeddings efficiently ban sakein.

---

## 7. Embeddings (Practical)
Text ko vectors mein convert karna — abhi sirf **concept** pata hai, code nahi likha (`GoogleGenerativeAIEmbeddings` jaisi classes).

---

## 8. Vector Stores
Embeddings ko store/search karna — Chroma, Pinecone, FAISS, Qdrant, ya simple `MemoryVectorStore`.

---

## 9. Retrievers
Vector DB se relevant chunks nikalna, RAG pipeline ka core part.

---

## 10. Streaming
Real-time typing-effect response (jaise ChatGPT). Abhi poora response ek saath aata hai.

---

## 11. Callbacks / LangSmith Tracing
Debugging aur monitoring ke liye — production mein zaroori.

---

## 12. Few-shot Prompting
LLM ko examples dekar behavior guide karna.

---

# 🎯 Abhi Tum Kya Bana Sakte Ho (LangChain-Only Skills)
- ✅ Kisi bhi LLM (Gemini/Groq) ko API se call karna
- ✅ System prompt se AI ka persona set karna
- ✅ Parameters tune karna (temperature, tokens, retries)
- ✅ Ek LLM se doosre LLM mein switch karna

# ❌ Abhi Tum Kya NAHI Bana Sakte
- ❌ Reusable prompt templates wala system
- ❌ Multi-step chain (LCEL) wala pipeline
- ❌ Structured/guaranteed JSON output
- ❌ Apna khud ka custom tool
- ❌ RAG/PDF-chatbot (document loading, chunking, embeddings, vector store — koi bhi nahi)
- ❌ Streaming response

---

## 📖 Recommended Order (Aage Kya Padhna Hai)
1. Prompt Templates
2. LCEL / Chains
3. Output Parsers
4. Custom Tools
5. Document Loaders + Text Splitters
6. Embeddings + Vector Stores + Retrievers (= RAG ban jayega)
7. Streaming
8. Callbacks/LangSmith