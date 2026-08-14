# 🤖 AI Full Course — Combined Master Notes
### LLM → Agent → LangChain → LangGraph (Complete Picture)

---

## 📊 Coverage Tracker (Honest, Rough Estimates)

```
LangChain:  ▓▓▓░░░░░░░░░░░░░░░░░  ~15-20%
LangGraph:  ▓▓▓▓▓░░░░░░░░░░░░░░░  ~25-30%
RAG:        ░░░░░░░░░░░░░░░░░░░░  0% (abhi padha nahi)
```
> Ye percentages **exact measurement nahi hain** — rough estimate hain based on kitne core features touch kiye. Detailed topic-wise breakdown alag files mein hai: `01_LangChain_Notes.md` aur `02_LangGraph_Notes.md`.

### 🎯 Abhi Kya Bana Sakte Ho (One Line)
> Single-tool, memory-wala AI agent (Jarvis-jaisa) jo real-time search kar sake — lekin PDF/RAG chatbot, custom tools, ya permanent memory wala system abhi nahi.

---

# SECTION A — THEORY FOUNDATION

## 1. Course Roadmap

```
Part 1: Level 0 (Backend) → Level 1 (Docker) → Level 2 (Redis) → Level 3 (System Design)
Part 2: Level 4 (AI Full Course) → Level 5 (Cloud Deployment) → Project

Level 4: LLM → LangChain → LangGraph → RAG → Vector DB   ← TUM YAHAN HO
```

---

## 2. Simple AI Response Generation (Direct LLM)

```
┌──────┐   Prompt    ┌─────┐   Response    ┌──────┐
│ User │ ──────────▶ │ LLM │ ────────────▶ │ User │
└──────┘             └─────┘               └──────┘
```

**LLM** = massive text data pe trained model jo human language samajhta aur generate karta hai.
Examples: GPT (OpenAI), Claude (Anthropic), Gemini (Google), Llama (Meta/Groq).

---

## 3. Problem Without Agent

| Problem | Explanation |
|---|---|
| ❌ No Realtime Data | Internet/live data access nahi |
| ❌ No Memory | Har request independent |
| ❌ No Tool/Action Access | Real actions nahi le sakta |
| ❌ No Multi-step Planning | Complex tasks khud se sequence mein nahi kar sakta |

---

## 4. AI Agent

```
Agent = LLM (Brain) + Tools + Memory + Action + Planning
```

### Agent Components
LLM (reasoning) → Memory → Tool calling → Planning → Execution loop → API/DB access

### LLM vs Agent

| Feature | LLM | Agent |
|---|---|---|
| Generates text | ✅ | ✅ |
| Uses tools/APIs | ❌ | ✅ |
| Makes decisions | Limited | ✅ |
| Multi-step tasks | Weak | Strong |
| Memory | Minimal | Persistent |
| Autonomous actions | ❌ | ✅ |

### Analogy
- **Direct LLM** = Smart insaan **band kamre mein** — sirf dimaag se jawab
- **AI Agent** = Wahi insaan, ab uske paas **phone (tools), notebook (memory), hands (actions)** hain

---

## 5. Frameworks Landscape

```
Agent Frameworks:
├── LangChain   → LLM calling, chains, RAG components
├── LangGraph   → Graph-based agent orchestration (LangChain ke upar)
├── LlamaIndex  → RAG-focused framework
├── CrewAI      → Multi-agent role-based framework
└── AutoGen     → Multi-agent conversation framework
```

---

## 6. What is LangChain?

```
LangChain connects: LLM + Memory + Tools + Vector DB + APIs
```
Banata hai: AI chatbots, RAG systems, AI agents, tool-calling apps, multi-step workflows.

---

## 7. What is LangGraph?

LangGraph, LangChain ke upar bana **graph-based framework** hai — complex, stateful, looping agent workflows ke liye.

**Kab use karein**: Jab agent ko loop chahiye, memory persist karni ho, ya conditional routing chahiye ho.

### Core Building Blocks
| Term | Matlab |
|---|---|
| Node | Ek step/function |
| Edge | Fixed connection do nodes ke beech |
| Conditional Edge | Condition check karke route decide karta hai |
| State | Poore graph ka shared data |
| Checkpointer | State save/persist karne ka mechanism |

---

# SECTION B — PRACTICAL CODE ANALYSIS

## 1. `.env` File

```env
GOOGLE_API_KEY="..."   # Gemini ke liye
GROQ_API_KEY="..."     # Groq (Llama 3.3) ke liye
TAVILY_API_KEY="..."   # Real-time web search ke liye
```

> LangChain wrappers apne env variable khud dhoondh lete hain — manual pass karne ki zaroorat nahi.

---

## 2. `package.json` Dependencies

| Package | Kaam |
|---|---|
| `@google/genai` | Raw Gemini SDK (bina LangChain) |
| `@langchain/core` | LangChain ka base package |
| `@langchain/google-genai` | Gemini ka LangChain wrapper |
| `@langchain/groq` | Groq ka LangChain wrapper |
| `@langchain/langgraph` | Graph-based agent workflow |
| `@langchain/tavily` | Real-time web search tool |
| `dotenv` | `.env` load karna |
| `express` | Backend server |
| `nodemon` | Dev auto-restart |

---

## 3. Without LangChain vs With LangChain vs With LangGraph

```javascript
// WITHOUT LangChain — raw SDK, one-shot, no memory, no tools
const ai = new GoogleGenAI({ apiKey: process.env.GEMINI_API_KEY })
const response = await ai.models.generateContent({ model, contents })
```

```javascript
// WITH LangGraph — full agent with memory + tools + loop
const llm = new ChatGroq({ model, temperature, maxTokens, maxRetries }).bindTools(tools)
const graph = new StateGraph(MessagesAnnotation)
    .addNode("agent", callLLM)
    .addNode("tools", toolNode)
    .addEdge("__start__", "agent")
    .addEdge("tools", "agent")
    .addConditionalEdges("agent", shouldContinue)
    .compile({ checkpointer: checkPointer })
```

---

## 4. Full Agent Graph Flow Diagram

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

### Component Breakdown

| Piece | Kaam |
|---|---|
| `TavilySearch` tool | Realtime web search karne ki power |
| `MemorySaver` (checkpointer) | Conversation memory save karta hai |
| `ToolNode` | Tools ko automatically execute karta hai |
| `.bindTools()` | LLM ko batata hai "ye tools available hain" |
| `callLLM` (agent node) | System prompt + messages ke saath LLM ko invoke karta hai |
| `shouldContinue` | Conditional edge — tool chahiye ya end karna hai, decide karta hai |
| `thread_id` | Alag conversations ko separate rakhta hai |

---

## 5. Real Scenario Walkthrough

**User pucho**: *"Aaj Delhi ka weather kaisa hai?"*

```
1. POST /ai, thread_id: "user123"
2. graph.invoke() → "agent" node run (callLLM)
3. LLM decide karta hai: "weather = realtime info, tool chahiye"
4. Response mein tool_calls hai → shouldContinue() → "tools"
5. ToolNode Tavily search execute karta hai → results milte hain
6. Graph wapas "agent" node pe jaata hai (.addEdge("tools", "agent"))
7. LLM tool result dekh ke final answer banata hai
8. shouldContinue() → tool_calls empty → "__end__"
9. Final answer return hota hai
```

**Agla message**: *"Mera naam Deveshh hai"* (same thread_id)
```
1. MemorySaver purani conversation load kar deta hai
2. Simple personal info hai → tool call nahi hoga
3. Directly "__end__" → answer return
```

---

## 6. Comparison Table — Teeno Approach

| Feature | Without LangChain | With LangChain (Simple) | With LangGraph (Agent) |
|---|---|---|---|
| Setup complexity | Simplest | Medium | Complex |
| Memory | ❌ | ❌ | ✅ MemorySaver |
| Tool calling | ❌ Manual | ⚠️ Manual loop | ✅ ToolNode automatic |
| Realtime data | ❌ | ⚠️ Manual | ✅ Bound tools |
| Multi-step loop | ❌ | ❌ | ✅ Graph loops |
| Conversation thread | ❌ | ❌ | ✅ thread_id |
| Best for | Simple Q&A | Basic chat apps | Real production agents |

---

# SECTION C — GLOSSARY (Sabhi Terms Ek Jagah)

| Term | Simple Definition |
|---|---|
| **LLM** | Large Language Model — text data pe trained AI jo language samajhta/generate karta hai |
| **Agent** | LLM + Tools + Memory + Action — autonomously tasks karne wala system |
| **Tool** | Ek function/API jo agent call kar sakta hai (search, calculator, etc.) |
| **Tool Calling** | LLM ka decide karna ki kaunsa tool kab use karna hai |
| **Node** (LangGraph) | Graph ka ek step/function |
| **Edge** (LangGraph) | Do nodes ke beech fixed connection |
| **Conditional Edge** | Condition check karke route decide karne wala edge |
| **State** (LangGraph) | Poore graph ka shared data (messages) |
| **Checkpointer** | State ko save/persist karne ka mechanism (memory) |
| **thread_id** | Conversation ko uniquely identify karne wala ID |
| **System Prompt** | AI ka role/behaviour define karne wala instruction |
| **Temperature** | Response ki randomness/creativity control karta hai (0=strict, 1=creative) |
| **RAG** | Retrieval Augmented Generation — LLM + external knowledge |
| **Embedding** | Text ko numbers/vectors mein convert karna |
| **Vector** | Numbers ki list jo semantic meaning represent karti hai |
| **Vector Database** | Semantic similarity search ke liye optimized database |
| **Chunking** | Bade documents ko chhote pieces mein todna |
| **Similarity Search** | Query se milte-julte (semantically) data dhoondhna |
| **LCEL** | LangChain Expression Language — `prompt \| llm \| parser` pipe syntax |
| **Prompt Template** | Reusable, variable-based prompt structure |
| **Output Parser** | LLM ka text output ko structured format mein convert karna |
| **Human-in-the-Loop** | Agent ko pause karke human approval lena |
| **Streaming** | Response ko real-time, piece-by-piece deliver karna |

---

# SECTION D — INTERVIEW Q&A (Jo Abhi Cover Kiya Uspe)

**Q1. LLM aur AI Agent mein kya difference hai?**
> LLM sirf text generate karta hai apni training knowledge se — koi tool use nahi kar sakta, memory nahi rakhta. Agent = LLM + Tools + Memory + Decision-making, jo autonomously multi-step tasks complete kar sakta hai.

**Q2. LangChain aur LangGraph mein kya difference hai?**
> LangChain LLM applications banane ke liye composable pieces deta hai (chains, prompts, tools) — linear/simple pipelines ke liye best. LangGraph, LangChain ke upar graph-based state machine deta hai — loops, conditional routing, aur complex multi-step agent workflows ke liye.

**Q3. Tumhara agent kab tool call karta hai aur kab nahi?**
> System prompt mein clearly define kiya hai — sirf realtime info (weather, news, search) ke liye tool call karo, warna simple conversation/memory-based answer do. Ye `shouldContinue` function ke through decide hota hai — agar LLM ke response mein `tool_calls` hain toh "tools" node pe jaate hain, warna graph end ho jaata hai.

**Q4. Memory kaise kaam karti hai tumhare agent mein?**
> `MemorySaver` (checkpointer) conversation state save karta hai, aur `thread_id` se decide hota hai kis conversation ki memory load karni hai. Lekin ye sirf **short-term/in-memory** hai — server restart hote hi memory chali jaati hai. Permanent memory ke liye Postgres/SQLite checkpointer chahiye (abhi nahi seekha).

**Q5. `.bindTools()` kya karta hai?**
> Ye LLM ko batata hai ki kaunse tools available hain. Isse LLM decide kar sakta hai ki kisi query ka jawab dene ke liye tool call karna hai ya nahi.

**Q6. Tumhara RAG ke baare mein kya knowledge hai?**
> *(Honest answer)*: Abhi sirf conceptual samajh hai — RAG kya hota hai, kyun chahiye, embeddings/vectors/similarity search ka flow. Practical implementation (document loaders, text splitters, vector store, retriever) abhi nahi kiya.

---

# SECTION E — WHAT YOU CAN / CANNOT BUILD (Honest)

### ✅ Abhi Bana Sakte Ho
- Basic LLM-powered API (system prompt se persona set karke)
- Memory-based chatbot (thread ke andar conversation yaad rakhne wala)
- Single-tool AI Agent (jaise Jarvis — search tool ke saath)
- Pre-built tool swap karna (Tavily → koi doosra tool)
- Model switch karna (Gemini ↔ Groq)

### ❌ Abhi Nahi Bana Sakte
- PDF/Document Chatbot (RAG) — koi document loading/chunking/embedding code nahi seekha
- Apna custom tool — sirf pre-built tools use kiye hain
- Structured JSON output — sirf plain text response milta hai
- Permanent/cross-session memory — restart pe memory gayab ho jaati hai
- Multi-agent system — ek hi agent node hai
- Prompt Templates / Chains (LCEL)
- Streaming response

---

## 🔜 Next Topics (Priority Order)
1. **RAG practically** — Document Loaders → Text Splitters → Embeddings → Vector Store → Retriever
2. **LangChain**: Prompt Templates, LCEL, Output Parsers, Custom Tools
3. **LangGraph**: Human-in-the-Loop, Long-term Memory, Persistent Checkpointer
4. **Level 5**: Cloud Deployment (AWS, CI/CD)

---

## 📂 Related Notes Files
- `01_LangChain_Notes.md` — LangChain deep-dive (covered + pending)
- `02_LangGraph_Notes.md` — LangGraph deep-dive (covered + pending)




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