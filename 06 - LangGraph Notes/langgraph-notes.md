# 🕸️ LangGraph — Complete Notes
### Covered Topics + Pending Roadmap

---

## 📊 Coverage Status
```
LangGraph Progress:  ▓▓▓▓▓░░░░░░░░░░░░░░░  ~25-30% (rough estimate, honest)
```
> Ye estimate hai — basic single-agent loop achhe se samajh aa gaya hai, production-level features baaki hain.

---

# ✅ PART 1 — Jo Tumne Ab Tak Cover Kiya

## 1. What is LangGraph?

LangGraph, LangChain ke upar bana **graph-based framework** hai jo complex, **stateful, looping agent workflows** banata hai.

**Kab use karte hain**:
- Jab agent ko **baar-baar decide** karna ho (loop chahiye)
- Jab **memory persist** karni ho across requests
- Jab **conditional routing** chahiye ho (agar X toh Y node, warna Z node)

---

## 2. Core Building Blocks

| Term | Matlab |
|---|---|
| **Node** | Ek step/function (jaise "agent" node ya "tools" node) |
| **Edge** | Do nodes ke beech fixed connection |
| **Conditional Edge** | Edge jo condition check karke route decide karta hai |
| **State** | Poore graph ka shared data (conversation messages) |
| **Checkpointer** | State ko save/persist karne ka mechanism |

---

## 3. StateGraph Setup (Practical)

```javascript
import { MessagesAnnotation, StateGraph, MemorySaver } from "@langchain/langgraph"
import { ToolNode } from "@langchain/langgraph/prebuilt"

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
| `new StateGraph(MessagesAnnotation)` | Graph banaya, "messages" based pre-built state schema use kiya |
| `.addNode(name, fn)` | Node register kiya |
| `.addEdge(from, to)` | Fixed connection banaya |
| `.addConditionalEdges(node, fn)` | Conditional routing function attach kiya |
| `.compile({ checkpointer })` | Graph final banaya, memory attach ki |

---

## 4. Graph Flow Diagram (Jo Banaya Tha)

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

**Flow explanation**:
1. Graph hamesha `agent` node se start hota hai
2. `agent` node LLM ko call karta hai
3. `shouldContinue` function check karta hai — tool chahiye ya nahi
4. Agar tool chahiye → `tools` node → wapas `agent` node (loop)
5. Agar nahi chahiye → `__end__` (graph ruk jaata hai)

---

## 5. ToolNode

```javascript
const toolNode = new ToolNode(tools)
```

**Kaam**: Pre-built node jo automatically decide karta hai kaunsa tool call karna hai, use execute karta hai, aur result return karta hai. Manual tool-calling logic likhne ki zaroorat nahi.

---

## 6. Agent Node (callLLM function)

```javascript
const callLLM = async (state) => {
    const response = await llm.invoke([
        { role: "system", content: "..." },
        ...state.messages
    ])
    return { messages: [response] }
}
```

- `state.messages` mein poori conversation history hoti hai (LangGraph automatically maintain karta hai)
- Response ko `messages` array mein return karte hain, LangGraph state mein append kar deta hai

---

## 7. Conditional Routing Logic (shouldContinue)

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

**Kaam**: LLM ka last response check karta hai — agar usme `tool_calls` hain, "tools" node pe route karo, warna graph khatam karo.

---

## 8. Memory — MemorySaver + thread_id

```javascript
const checkPointer = new MemorySaver()

// Route mein use:
await graph.invoke(
    { messages: [{ role: "user", content: input }] },
    { configurable: { thread_id: "user123" } }
)
```

| Concept | Matlab |
|---|---|
| `MemorySaver` | Conversation state ko save karta hai (in-memory, **server restart pe gayab ho jaata hai**) |
| `thread_id` | Har conversation ko alag rakhta hai — 2 users ke liye alag `thread_id` do taaki memory mix na ho |

> **Important honesty note**: `MemorySaver` sirf **short-term/thread memory** hai — permanent database-backed memory nahi. Server restart hote hi sab memory chali jaati hai.

---

# ❌ PART 2 — Jo Abhi Baaki Hai (Real LangGraph Roadmap)

## 1. Human-in-the-Loop (`interrupt()`)
Agent ko **pause** karke human approval lena, phir continue karna. Production mein bahut important — jaise "agent payment karne se pehle human se confirm le".

---

## 2. Long-term Memory (Store)
`MemorySaver` sirf ek thread ke andar kaam karta hai. **Long-term memory (`Store`)** cross-session hoti hai — user ke baare mein hamesha yaad rakhna (naya thread ho tab bhi).

---

## 3. Multi-Agent Systems
Multiple agents jo aapas mein coordinate karte hain (Supervisor pattern) — ek agent doosre ko task delegate kare.

---

## 4. Subgraphs
Ek graph ke andar doosra graph nest karna — modular, reusable agent design.

---

## 5. Custom State Schema
Abhi sirf pre-built `MessagesAnnotation` use kiya hai. `Annotation.Root` se apna khud ka complex state bana sakte ho (sirf messages nahi, custom fields bhi).

---

## 6. Streaming Graph Execution
Real-time step-by-step output dikhana jab graph chal raha ho (abhi poora result ek saath aata hai).

---

## 7. Parallel Node Execution
Ek saath multiple nodes run karna (branching) — speed ke liye important.

---

## 8. Persistent Checkpointer (Postgres/SQLite)
`MemorySaver` sirf in-memory hai. Production mein **Postgres ya SQLite checkpointer** use karte hain taaki restart ke baad bhi memory bachi rahe.

---

## 9. Time Travel / State History
Purani graph state pe wapas jaana, debug karna, kya-kya decisions liye gaye wo dekhna.

---

## 10. Command / Send API
Runtime pe dynamic routing decide karna (advanced control flow).

---

# 🎯 Abhi Tum Kya Bana Sakte Ho (LangGraph-Only Skills)
- ✅ Single-agent, single-tool AI Agent (jaise Jarvis)
- ✅ Conversation ke andar memory rakhne wala chatbot (thread-based)
- ✅ Agent jo khud decide kare — tool use karna hai ya nahi
- ✅ Basic Agent ↔ Tools loop (ReAct pattern)

# ❌ Abhi Tum Kya NAHI Bana Sakte
- ❌ Human approval wala safe agent (interrupt)
- ❌ Permanent/cross-session memory wala system
- ❌ Multi-agent coordination (jaise supervisor + workers)
- ❌ Modular subgraphs
- ❌ Real-time streaming agent responses
- ❌ Production-grade persistent memory (Postgres)

---

## 📖 Recommended Order (Aage Kya Padhna Hai)
1. Custom State Schema (`Annotation.Root`)
2. Human-in-the-Loop (`interrupt()`)
3. Long-term Memory (`Store`)
4. Persistent Checkpointer (Postgres/SQLite)
5. Streaming
6. Multi-Agent Systems + Subgraphs