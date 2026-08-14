# 💻 Code Explanation — LangGraph AI Agent (Full Breakdown)
### Tumhare Diye Gaye Real Code Ka Line-by-Line Analysis

---

## 📄 Poora Code (Reference)

```javascript
import express from "express"
import dotenv from "dotenv"
import { GoogleGenAI } from "@google/genai"
import { ChatGoogleGenerativeAI } from "@langchain/google-genai"
import { ChatGroq } from "@langchain/groq"
import { Annotation, MemorySaver, MessagesAnnotation, StateGraph } from "@langchain/langgraph"
import { ToolNode } from "@langchain/langgraph/prebuilt";
import { TavilySearch } from "@langchain/tavily";

dotenv.config()
const app = express()
const port = 5000
app.use(express.json())

//without Langchain
// const ai = new GoogleGenAI({
//     apiKey: process.env.GEMINI_API_KEY
// })
// app.post("/ai", async (req, res) => {
//     const { input } = req.body
//     const response = await ai.models.generateContent({
//         model: "gemini-3.5-flash",
//         contents: [
//             {
//                 role: "system",
//                 parts: [{ text: "you are a assistant and your name is jarvis.if you don't know the answer then don't give incorrect answer" }]
//             },
//             {
//                 role: "user",
//                 parts: [{ text: input }]
//             }
//         ]
//     })
//     return res.status(200).json({ "ai:": response.text })
// })

//with langchain
const tool = new TavilySearch({
    maxResults: 5,
    topic: "general",
});

const checkPointer = new MemorySaver()
const tools = [tool]
const toolNode = new ToolNode(tools)

const llm = new ChatGroq({
    model: "llama-3.3-70b-versatile",
    temperature: 0.7,
    maxTokens: 100,
    maxRetries: 2
}).bindTools(tools)

const callLLM = async (state) => {
    console.log("state:", state)
    const response = await llm.invoke([
        {
            role: "system",
            content: `You are Jarvis AI assistant
Use conversation memory first.
Only use tools when the answer requires
external real-time information like:
weather, news, web search, stock prices etc.
Do NOT call tools for simple conversation,
memory-based questions, greetings,
or personal context`
        },
        ...state.messages
    ])
    return { messages: [response] }
}

const shouldContinue = async (state) => {
    const lastMessage = state.messages[state.messages.length - 1]
    if (lastMessage.tool_calls.length > 0) {
        return "tools"
    } else {
        return "__end__"
    }
}

const graph = new StateGraph(MessagesAnnotation)
    .addNode("agent", callLLM)
    .addNode("tools", toolNode)
    .addEdge("__start__", "agent")
    .addEdge("tools", "agent")
    .addConditionalEdges("agent", shouldContinue)
    .compile({ checkpointer: checkPointer })

app.post("/ai", async (req, res) => {
    const { input } = req.body
    const response = await graph.invoke(
        {
            messages: [
                {
                    role: "user",
                    content: input
                }
            ]
        },
        { configurable: { thread_id: "user123" } }
    )
    console.log(response.messages)
    return res.status(200).json({ "ai:": response.messages[response.messages.length - 1].content })
})

app.get("/", (req, res) => {
    return res.json({ message: "hello from level4" })
})

app.listen(port, () => {
    console.log("server started")
})
```

---

## 1️⃣ `.env` File — Kya aur Kyun

```env
GOOGLE_API_KEY="add your"
GROQ_API_KEY="add your"
TAVILY_API_KEY="add your"
```

| Key | Kis Liye |
|---|---|
| `GOOGLE_API_KEY` | Gemini model call karne ke liye (raw SDK wale commented part mein) |
| `GROQ_API_KEY` | Groq ka LLM (Llama 3.3) call karne ke liye — LangChain automatically isko uthata hai |
| `TAVILY_API_KEY` | Tavily search tool ke liye — real-time web search ke liye |

> **Important**: LangChain/LangGraph packages apne corresponding env variable **khud dhoondh lete hain**, manually pass karne ki zaroorat nahi.

---

## 2️⃣ `package.json` Dependencies — Ek-Ek Karke

```json
{
  "dependencies": {
    "@google/genai": "^2.8.0",
    "@langchain/core": "^1.1.49",
    "@langchain/google-genai": "^2.1.31",
    "@langchain/groq": "^1.2.1",
    "@langchain/langgraph": "^1.4.2",
    "@langchain/tavily": "^1.2.0",
    "dotenv": "^17.4.2",
    "express": "^5.2.1",
    "nodemon": "^3.1.14"
  }
}
```

| Package | Kaam |
|---|---|
| `@google/genai` | Raw Gemini SDK (bina LangChain ke direct call) |
| `@langchain/core` | LangChain ka base/core package — sabki dependency |
| `@langchain/google-genai` | Gemini ko LangChain ke through call karne ka wrapper (`ChatGoogleGenerativeAI`) |
| `@langchain/groq` | Groq ko LangChain ke through call karne ka wrapper (`ChatGroq`) |
| `@langchain/langgraph` | Graph-based agent workflow banane ke liye (Node, Edge, StateGraph, Memory) |
| `@langchain/tavily` | Real-time web search tool integration |
| `dotenv` | `.env` file read karne ke liye |
| `express` | Backend server/API banane ke liye |
| `nodemon` | Dev mode mein auto-restart server |

---

## 3️⃣ Imports — Kaunsa Kis Liye Aaya

```javascript
import express from "express"
import dotenv from "dotenv"
import { GoogleGenAI } from "@google/genai"
import { ChatGoogleGenerativeAI } from "@langchain/google-genai"
import { ChatGroq } from "@langchain/groq"
import { Annotation, MemorySaver, MessagesAnnotation, StateGraph } from "@langchain/langgraph"
import { ToolNode } from "@langchain/langgraph/prebuilt"
import { TavilySearch } from "@langchain/tavily"
```

| Import | Role |
|---|---|
| `express` | Server banane ke liye |
| `dotenv` | `.env` load karne ke liye |
| `GoogleGenAI` | Raw SDK (without-LangChain wale commented part mein use hua) |
| `ChatGoogleGenerativeAI` | Gemini ko LangChain se use karne ka option — **is code mein import hua hai but actively use nahi hua** (kyunki final agent Groq use kar raha hai) |
| `ChatGroq` | Groq ka fast LLM (Llama 3.3 70B) — **yehi actual LLM hai jo use ho raha hai** |
| `Annotation, MemorySaver, MessagesAnnotation, StateGraph` | LangGraph ke core tools — graph banane, state define karne, aur memory ke liye |
| `ToolNode` | LangGraph ka pre-built node jo tools ko execute karta hai |
| `TavilySearch` | Real-time internet search karne wala tool |

---

## 4️⃣ Without LangChain (Commented Part) — Kya Hai Aur Kyun Limited Hai

```javascript
const ai = new GoogleGenAI({ apiKey: process.env.GEMINI_API_KEY })

app.post("/ai", async (req, res) => {
    const { input } = req.body
    const response = await ai.models.generateContent({
        model: "gemini-3.5-flash",
        contents: [
            { role: "system", parts: [{ text: "you are a assistant..." }] },
            { role: "user", parts: [{ text: input }] }
        ]
    })
    return res.status(200).json({ "ai:": response.text })
})
```

**Line-by-line**:
- `new GoogleGenAI({ apiKey: ... })` → Raw SDK client banaya, **manually** API key pass ki (LangChain jaisa automatic nahi)
- `ai.models.generateContent(...)` → Direct Gemini API call, sirf ek response aata hai
- `contents` array mein `role: "system"` aur `role: "user"` manually format karne pade — LangChain isko simplify kar deta

**Limitation**: Ye sirf **ek baar** LLM ko call karta hai. No memory, no tools, no loop. Agar realtime data chahiye ho (weather), ye system fail ho jayega ya LLM hallucinate karega. Isi wajah se system prompt mein safety patch likha hai:
> *"if you don't know the answer then don't give incorrect answer"* — ye ek **patch** hai, real solution nahi. Real solution hai Agent + Tools dena.

---

## 5️⃣ With LangChain + LangGraph — Step-by-Step

### 🔹 STEP 1: Tool Banaya (Real-time Search)

```javascript
const tool = new TavilySearch({
    maxResults: 5,
    topic: "general",
});
```

**Kyun**: LLM ko internet search karne ki power deta hai. Jab realtime info chahiye (weather, news, stock price), LLM isi tool ko call karega.
- `maxResults: 5` → max 5 search results
- `topic: "general"` → general web search (specific news/finance topic nahi)

---

### 🔹 STEP 2: Memory Setup

```javascript
const checkPointer = new MemorySaver()
```

**Kyun**: LangGraph ka **memory mechanism**. Iske bina, har request independent hoti. `MemorySaver` conversation state ko save karta hai taaki agla message aane par LLM ko pura context mile.

> Ye "No Memory" problem ka solution hai jo Direct LLM mein tha.

---

### 🔹 STEP 3: Tools Ko Node Banaya

```javascript
const tools = [tool]
const toolNode = new ToolNode(tools)
```

**Kyun**: LangGraph mein har cheez ek **Node** hoti hai. `ToolNode` ek pre-built node hai jo automatically decide karta hai kaunsa tool call karna hai aur usko execute karke result wapas deta hai. Manual tool-calling logic likhne ki zaroorat nahi.

---

### 🔹 STEP 4: LLM Banaya aur Tools Bind Kiye

```javascript
const llm = new ChatGroq({
    model: "llama-3.3-70b-versatile",
    temperature: 0.7,
    maxTokens: 100,
    maxRetries: 2
}).bindTools(tools)
```

| Parameter | Matlab |
|---|---|
| `model` | Groq ka Llama 3.3 70B model (bahut fast inference speed) |
| `temperature: 0.7` | Response mein thoda creativity/randomness (0 = strict, 1 = creative) |
| `maxTokens: 100` | Response ki max length limit |
| `maxRetries: 2` | Fail hone par 2 baar retry karega |
| `.bindTools(tools)` | **Sabse important** — LLM ko batata hai "tumhare paas ye tools available hain, zaroorat pade toh use karo" |

**Kyun `.bindTools()` zaroori hai**: Bina isske, LLM ko pata hi nahi chalega ki `tool` (Tavily search) exist karta hai. Bind karne ke baad, LLM khud decide karega — "iska jawab dene ke liye mujhe search karna padega" ya "nahi, ye mujhe pehle se pata hai".

---

### 🔹 STEP 5: Agent Node Define Kiya (callLLM)

```javascript
const callLLM = async (state) => {
    console.log("state:", state)
    const response = await llm.invoke([
        {
            role: "system",
            content: `You are Jarvis AI assistant
Use conversation memory first.
Only use tools when the answer requires
external real-time information like:
weather, news, web search, stock prices etc.
Do NOT call tools for simple conversation,
memory-based questions, greetings,
or personal context`
        },
        ...state.messages
    ])
    return { messages: [response] }
}
```

**Kya kar raha hai**:
1. `state` mein poori conversation history hoti hai (LangGraph automatically maintain karta hai)
2. System prompt LLM ko **rules** deta hai — "sirf tab tool use karo jab realtime data chahiye ho, warna memory/conversation se hi jawab do"
3. `...state.messages` — pichle saare messages spread kar diye, taaki LLM ko **poora context** mile
4. Response ko wapas `messages` array mein return kar diya (LangGraph isko state mein append kar dega)

**Kyun ye important hai**: Ye system prompt hi decide karta hai ki LLM **overuse** na kare tools ko. Agar ye na likha hota, LLM har chhoti baat pe bhi search tool call kar sakta tha (slow + costly).

---

### 🔹 STEP 6: Decision Logic (shouldContinue)

```javascript
const shouldContinue = async (state) => {
    const lastMessage = state.messages[state.messages.length - 1]
    if (lastMessage.tool_calls.length > 0) {
        return "tools"
    } else {
        return "__end__"
    }
}
```

**Kya kar raha hai**: Ye function check karta hai — LLM ka jo aakhri response aaya, usme koi **tool_call** request hai kya?
- Agar haan → `"tools"` node pe bhejo (tool execute karo)
- Agar nahi → `"__end__"` (matlab final answer aa gaya, graph khatam karo)

**Kyun ye zaroori hai**: Yehi wo **Conditional Edge** hai jo notes mein diagram mein tha. Isi se graph decide karta hai — loop continue karna hai ya rukna hai.

---

### 🔹 STEP 7: Graph Banaya (StateGraph)

```javascript
const graph = new StateGraph(MessagesAnnotation)
    .addNode("agent", callLLM)
    .addNode("tools", toolNode)
    .addEdge("__start__", "agent")
    .addEdge("tools", "agent")
    .addConditionalEdges("agent", shouldContinue)
    .compile({ checkpointer: checkPointer })
```

| Line | Matlab |
|---|---|
| `new StateGraph(MessagesAnnotation)` | Naya graph banaya jiska state-schema "messages" based hai (pre-built LangGraph schema) |
| `.addNode("agent", callLLM)` | "agent" naam ka node banaya jo `callLLM` function run karega |
| `.addNode("tools", toolNode)` | "tools" naam ka node banaya jo tool execute karega |
| `.addEdge("__start__", "agent")` | Graph hamesha **agent** node se shuru hoga |
| `.addEdge("tools", "agent")` | Tool execute hone ke baad **hamesha wapas agent** pe jayega (taaki LLM tool ka result dekh ke final answer de) |
| `.addConditionalEdges("agent", shouldContinue)` | Agent ke baad, `shouldContinue` function decide karega — tools pe jaana hai ya end karna hai |
| `.compile({ checkpointer: checkPointer })` | Graph ko final banaya, memory attach ki |

### 📊 Graph Flow (Visual)

```
        ┌─────────┐
 START ─▶  agent   │◀────────┐
        └────┬────┘         │
             │ (conditional) │
       ┌─────┴─────┐        │
     tools?       no tools  │
       │           │        │
       ▼           ▼        │
   ┌───────┐     END        │
   │ tools │─────────────────┘
   └───────┘
```

---

### 🔹 STEP 8: Express Route (Graph Ko Call Karna)

```javascript
app.post("/ai", async (req, res) => {
    const { input } = req.body
    const response = await graph.invoke(
        {
            messages: [{ role: "user", content: input }]
        },
        { configurable: { thread_id: "user123" } }
    )
    return res.status(200).json({ "ai:": response.messages[response.messages.length - 1].content })
})
```

| Part | Kyun |
|---|---|
| `graph.invoke(...)` | Poore graph ko run karta hai (agent → shayad tools → agent → end) |
| `messages: [{ role: "user", content: input }]` | User ka naya message state mein daala |
| `thread_id: "user123"` | **Sabse important** — batata hai kis "conversation thread" ki memory use karni hai. Same `thread_id` rahe toh LangChain automatically pichli memory load kar lega |
| `response.messages[last].content` | Graph ke final message (jo LLM ne generate kiya) ka content nikal ke bhej diya |

> **thread_id ka concept**: Agar 2 alag users hain, unko alag `thread_id` do (jaise `user123`, `user456`), taaki unki memory mix na ho.

---

## 6️⃣ Poora Scenario — Real Example Se Samjho

**User pucho**: *"Aaj Delhi ka weather kaisa hai?"*

```
1. Request /ai route pe aata hai, thread_id: "user123"
2. graph.invoke() call hota hai → "agent" node run hota hai (callLLM)
3. LLM system prompt padhta hai: "realtime info ke liye tool use karo"
4. LLM decide karta hai: "weather = realtime info, mujhe search tool chahiye"
5. LLM response mein tool_calls hota hai → shouldContinue() check karta hai
6. tool_calls.length > 0 → "tools" node pe route hota hai
7. ToolNode Tavily search execute karta hai → results milte hain
8. Graph wapas "agent" node pe jaata hai (.addEdge("tools", "agent"))
9. LLM ab tool ka result dekh ke final answer banata hai
10. shouldContinue() check karta hai → is baar tool_calls empty hai
11. "__end__" → graph ruk jaata hai, final answer return hota hai
```

**User pucho (agla message)**: *"Mera naam Deveshh hai"*
```
1. Same thread_id "user123" use hota hai
2. MemorySaver purani conversation load kar deta hai
3. LLM ko pata chal jaata hai user ne pehle weather pucha tha
4. Simple greeting/personal info hai → tool call nahi hoga
5. Directly "__end__" → answer return
```

---

## 7️⃣ Ye Code Kya Prove Karta Hai (Recap)

Ye code exactly wahi answer hai jo humne pehle discuss kiya tha — **"Direct LLM ki jagah Agent/LangGraph kyun use karte hain"**:

| Direct LLM Problem | Is Code Mein Solution |
|---|---|
| No realtime data | `TavilySearch` tool + `.bindTools()` |
| No memory | `MemorySaver` + `thread_id` |
| No multi-step decision | `StateGraph` + `shouldContinue` (conditional loop) |
| No tool execution | `ToolNode` |