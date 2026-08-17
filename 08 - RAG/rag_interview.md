# 📝 Self-Test Quiz
### Pehle Khud Try Karo, Phir Neeche Answers Dekho!

> Instructions: Har question ka answer khud sochke likho (paper pe ya mann mein), phir scroll karke check karo. Jo galat ho jaaye, uska topic dubara padho.

---

## 🟢 EASY Level (15 Questions)

1. LLM ka full form kya hai?
2. Agent ka formula kya hai? (Agent = ?)
3. Direct LLM ki 2 sabse badi limitations kya hain?
4. Embedding kya karta hai (ek line mein)?
5. Vector kya hota hai?
6. RAG ka full form kya hai aur har letter ka matlab?
7. Vector Database, Normal Database se kaise alag hai?
8. Chunking kyun ki jaati hai?
9. LangChain kya connect karta hai? (4 cheezein)
10. LangGraph kis liye use hota hai?
11. `temperature` parameter kya control karta hai?
12. System role aur Human role mein kya difference hai?
13. `.env` file mein API keys kyun rakhte hain, seedha code mein kyun nahi?
14. RAG mein "Grounding" ka kya matlab hai?
15. Similarity Search kya karta hai?

---

## 🟡 MEDIUM Level (10 Questions)

16. `.bindTools()` function ka kaam kya hai?
17. `MemorySaver` aur `thread_id` saath mein kaise kaam karte hain?
18. LangGraph mein "Conditional Edge" kya hota hai, example do
19. `chunkOverlap` kyun zaroori hai? Real example se samjhao
20. `TaskType.RETRIEVAL_DOCUMENT` aur `RETRIEVAL_QUERY` mein kya difference hai?
21. `ToolNode` kya kaam karta hai LangGraph mein?
22. Tumhare RAG code mein `upload()` function ka kya problem tha?
23. RAG pipeline ke do phases kaunse hain?
24. Agar user PDF mein na hone wala sawaal pucche, RAG system kya karega (agar sahi prompt ho)?
25. `QdrantVectorStore.fromExistingCollection()` mein "existing" kyun likha hai — iska matlab kya hai?

---

## 🔴 HARD Level (5 Questions — Concept Only, OK Agar Na Aaye)

26. Re-ranking kya hota hai aur ye similarity search se kaise alag hai?
27. Hybrid Search kya combine karta hai?
28. Agentic RAG normal RAG se kaise alag hai?
29. RAG Evaluation mein "Faithfulness" metric kya check karti hai?
30. `MemorySaver` production mein kyun kaafi nahi hai — iski jagah kya use karte hain?

---
---
---

# ✅ ANSWERS

## EASY Level

1. **LLM** = Large Language Model — massive text data pe trained AI jo language samajhta/generate karta hai
2. **Agent = LLM (Brain) + Tools + Memory + Action + Planning**
3. **(a)** No realtime data access, **(b)** No memory (har request independent)
4. Embedding text ko **numbers/vectors** mein convert karta hai, taaki computer uska meaning samajh sake
5. Vector = **numbers ki list** jo kisi text ka semantic meaning represent karti hai (jaise `[0.21, -0.82, 0.55]`)
6. **R**etrieval (dhoondhna) + **A**ugmented (badhaana) + **G**eneration (banana)
7. Normal DB **exact match** search karta hai (SQL), Vector DB **semantic/meaning-based** search karta hai
8. Kyunki bada document LLM ko ek saath dena inefficient hai — chhote chunks mein todke sirf relevant parts use karte hain
9. LLM + Memory + Tools + Vector DB + APIs
10. Jab agent ko **loop/multi-step decisions**, **persistent memory**, aur **conditional routing** chahiye ho
11. Response ki **randomness/creativity** — 0 = strict/factual, 1 = creative/random
12. **System** = AI ka behaviour/rules define karta hai; **Human** = user ka actual message
13. Security ke liye (keys code mein hardcode nahi karte, `.env` git mein commit nahi hoti) + easy configuration change
14. LLM ko **sirf diye gaye context tak seemit rakhna** — bahar ka knowledge use na karne dena, taaki hallucination na ho
15. User ke query ko embed karke, Vector DB mein **sabse semantically similar chunks** dhoondhna

## MEDIUM Level

16. LLM ko batata hai ki **kaunse tools available hain**, taaki LLM decide kar sake kab tool call karna hai
17. `MemorySaver` conversation state save karta hai; `thread_id` batata hai **kis conversation ki memory load karni hai** — same `thread_id` = same memory
18. Ek edge jo **condition check karke** decide karta hai kahan route karna hai. Example: `shouldContinue` — agar `tool_calls` hain toh "tools" node, warna "__end__"
19. Overlap na ho toh important context **do chunks ke beech kat sakta hai**. Example: "recipe 180°C pe bake karo" — agar ye line split ho jaaye do chunks mein, matlab hi kho sakta hai. Overlap se thoda text repeat hota hai, continuity bani rehti hai
20. `RETRIEVAL_DOCUMENT` = jab data **store** kar rahe ho; `RETRIEVAL_QUERY` = jab **search query** embed kar rahe ho — dono ke liye optimal embeddings alag hoti hain
21. Automatically decide karta hai kaunsa tool call karna hai, use **execute** karta hai, aur result return karta hai
22. Ye function kahin **call hi nahi ho raha** code mein — koi route/trigger nahi hai, isliye PDF kabhi index nahi hoga jab tak manually call na ho
23. **Indexing** (PDF → chunks → embeddings → vector DB, one-time) aur **Retrieval + Generation** (query → search → context → LLM answer, har request pe)
24. System prompt ke rules follow karega aur bolega: **"I don't know from uploaded PDF"** — galat jawab nahi banayega (agar grounding prompt sahi likha ho)
25. Matlab collection **pehle se Qdrant mein bani honi chahiye** — ye function naya collection nahi banata, sirf existing se connect karta hai

## HARD Level

26. Re-ranking similarity search ke top results ko ek **doosre, zyada powerful model** se dobara sort karta hai, taaki sabse relevant chunk upar aaye (similarity search sirf ek pehla filter hai, kabhi perfect order nahi deta)
27. **Semantic (meaning-based) search + Keyword (exact match/BM25) search** — dono combine karke behtar results
28. Agentic RAG mein **Agent khud decide karta hai** kab retrieve karna hai, kitni baar, aur kahan se — normal RAG mein retrieval hamesha fixed/automatic hota hai har query pe
29. Check karti hai ki LLM ka answer **actually diye gaye context se match** karta hai ya nahi (kahin LLM ne apni taraf se kuch add toh nahi kiya)
30. `MemorySaver` sirf **in-memory** hai — server restart hote hi memory chali jaati hai. Production mein **Postgres/SQLite checkpointer** use karte hain jo permanent storage deta hai

---

## 📊 Apna Score Nikaalo
```
EASY (15):    ___/15
MEDIUM (10):  ___/10
HARD (5):     ___/5

Total: ___/30
```

**Scoring guide**:
- 25+/30 → Foundation bahut strong hai, practical projects shuru karo
- 18-24/30 → Achha hai, EASY+MEDIUM dubara revise karo jahan galti hui
- <18/30 → Concerned mat ho, ye normal hai — Master Notes files dubara padho, especially jahan miss hua