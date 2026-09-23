RAG (Retrieval-Augmented Generation) — Complete Notes
Based on pdfRag.agent.js — apna code, apni terminology, shuru se aakhir tak.


## 1. RAG Hai Kya? (Sabse Pehle Ye Samjho)

Problem: LLM ek baar mein limited text hi padh sakta hai (context window), aur agar tum poori 100-page PDF usko de do, ye mehenga bhi hoga aur LLM confuse bhi ho sakta hai.

RAG ka solution: Poori PDF LLM ko mat do. Pehle relevant hissa dhoondo (Retrieval), fir sirf wahi hissa LLM ko do jawab banane ke liye (Generation). Beech mein jo "dhoondne" ka kaam hai, usse Augmented kehte hain — matlab LLM ke jawab ko extra info se "behtar" banana.

Full form:

R = Retrieval        → relevant info dhoondo
A = Augmented         → us info se LLM ka jawab behtar banao
G = Generation         → LLM final jawab likhe
Analogy (kisी ko samjhana ho to ye bolo):

"Socho tumse koi poochhe 'is 500-page kitab mein page 340 pe kya likha hai'. Tum poori kitab yaad nahi karoge — tum index dekhoge, sahi page dhoondoge, sirf wahi paragraph padhoge, aur jawab doge. RAG bhi yahi karta hai — poora document 'yaad' nahi karta, sirf relevant paragraph dhoondta hai, aur usी se jawab banata hai."


## 2. Poora Pipeline — Ek Nazar Mein

PDF Upload
   |
   v
[1] Text Extraction     -> PDF se raw text nikalna
   |
   v
[2] Chunking            -> text ko chhote tukdo mein todna
   |
   v
[3] Embedding            -> har chunk ko "numbers" (vector) mein badalna
   |
   v
[4] Vector Store          -> un numbers ko database (Qdrant) mein save karna
   |
   v
[5] Retrieval             -> user ka sawaal aaya, usse milte-julte chunks dhoondo
   |
   v
[6] Augmentation           -> mile hue chunks + sawaal, ek prompt mein jodo
   |
   v
[7] Generation              -> LLM ko do, jawab milega
Ye poora pipeline ek "official naam" se jaana jaata hai: iसे RAG Pipeline ya Retrieval Pipeline kehte hain. Jab koi poochhe "ye kya bana rahe ho", bol sakte ho: "Maine ek RAG pipeline banaya hai jo PDF ko chunk karke, vector-search se relevant context nikaal ke LLM ko deta hai."


## 3. Apna Code — Shuru Se Aakhir Tak (Sochne Ka Process)

Ye section socho jaise tum khud se ye code likh rahe ho — har line ke baad "agli line kyun aayi" wo bhi samjhaya hai.


### Step 0 — Socho: "Mujhe kya chahiye final result mein?"

Pehla sawaal jo dimaag mein aana chahiye:

"User ek PDF degा aur ek sawaal poochega. Mujhe uska sahi jawab dena hai, wo bhi sirf usी PDF ke content se — LLM ko poori PDF nahi de sakta."

Isse decide hota hai: mujhe chunking + search + LLM — teeno chahiye honge.


### Step 1 — File Padhna

const buffer = fs.readFileSync(state.file.path);
Soch: "File disk pe hai (Multer ne save ki thi), mujhe usse RAM mein laana hai taaki process kar sakun."

Ye kya return karta hai: Ek Buffer — raw binary data (jaisa multer notes mein bataya tha).

Official naam: Isko "raw file read" ya "binary read" kehte hain — abhi ye "text" nahi hai, sirf raw bytes hain.


### Step 2 — PDF Se Text Nikalna

const pdf = new PDFParse({ data: buffer });
const result = await pdf.getText();
const text = result.text;
Soch: "Buffer to raw hai, mujhe usme se actual PADHNE-LAYAK text chahiye — jo LLM samajh sake."

Official naam: Isko "Text Extraction" kehte hain — PDF ek binary format hai (images, fonts, layout sab mixed), isse plain-text nikaalna ek alag kaam hai, jo pdf-parse library karti hai.

💡 Yaad rakhne wali cheez: .getText() async hai (Promise return karta hai), isliye await chahiye — ye wahi bug tha jo humne fix kiya.


### Step 3 — Text Ko Chunks Mein Todna

const splitter = new RecursiveCharacterTextSplitter({
  chunkSize: 1000,
  chunkOverlap: 200,
});
const docs = await splitter.createDocuments([text]);
Soch: "Poora text ek saath LLM ko nahi de sakta (bahut bada hai). Mujhe isko chhote pieces mein todna hoga."

Official naam: Isko "Chunking" kehte hain, aur har chhote piece ko "chunk" ya "document" kehte hain (isilye variable ka naam docs hai — LangChain ki terminology mein har chunk ek "Document" object hota hai).

chunkSize: 1000 ka matlab: Har chunk 1000 characters ka hoga (roughly ~150-200 words).

chunkOverlap: 200 ka matlab — ye sabse important concept hai: Har chunk ka aakhri 200 characters, agle chunk ke shuruaati 200 characters ke saath repeat honge.

Kyun overlap chahiye (bina overlap ke kya problem hoti):

Bina overlap ke:
Chunk 1: "...Ram ek accha ladka hai jo humesha"
Chunk 2: "school time pe pahunchta hai..."

Agar user poochhe "Ram kaisa hai school ke time ke baare mein?"
→ Dono chunks alag-alag hain, poora context kisi ek chunk mein nahi mila!
Overlap ke saath:
Chunk 1: "...Ram ek accha ladka hai jo humesha school time pe"
Chunk 2: "humesha school time pe pahunchta hai..."

Ab dono chunks mein zaroori context maujood hai, sentence beech mein nahi tuta.
Rule of thumb: Overlap = chunkSize ka 15-25% rakha jaata hai (1000 mein 200 = 20%, tumhara ratio sahi hai).


### Step 4 — Chunks Ko "Numbers" Mein Badalna

// embeddings.js
export const embeddings = new GoogleGenerativeAIEmbeddings({
  model: "gemini-embedding-001",
});
Soch: "Database mein text ko 'similarity' se search karna hai — lekin computer text ki 'similarity' seedha nahi samajh sakta, use numbers chahiye."

Official naam: Isko "Embedding" kehte hain, aur jo model ye kaam karta hai use "Embedding Model" kehte hain. Result (numbers ki list) ko "vector" kehte hain.

Kaise kaam karta hai (analogy): Har sentence ko ek GPS coordinate mil jaata hai. Similar-meaning sentences ke coordinates paas-paas hote hain.

"React ek library hai"        -> [0.23, -0.41, 0.88, ...]
"React frontend ke liye hai"  -> [0.25, -0.38, 0.85, ...]   (paas-paas, similar meaning)
"Aaj mausam accha hai"        -> [-0.91, 0.12, -0.55, ...]   (door, alag meaning)

### Step 5 — Vector Database Mein Store Karna

// vectorDb.js
export const vectorStore = async (docs, collectionName) => {
  return await QdrantVectorStore.fromDocuments(docs, embeddings, {
    url: process.env.QDRANT_URL,
    collectionName,
  });
};
Soch: "Har chunk ka vector ban gaya, ab mujhe inhe kahi STORE karna hai taaki baad mein search kar sakun."

Official naam: Isko "Vector Store" ya "Vector Database" kehte hain. Qdrant iska ek specific product/tool hai (jaise MongoDB normal database hai, Qdrant vector-database hai). fromDocuments() — ye method naya collection banata hai + chunks insert karta hai (ye wahi fix tha jo humne kiya — fromExistingCollection galat tha kyunki wo sirf purana collection padhta hai, naya nahi banata).

Har PDF ke liye naya collection kyun (pdf-${Date.now()}): Taaki ek PDF ke chunks, doosri PDF ke chunks se mix na ho — har upload isolated rehta hai.


### Step 6 — Relevant Chunks Dhoondna

const relevantDocs = await store.similaritySearch(state.prompt, 5);
Soch: "User ne sawaal poocha, ab mujhe us sawaal se sabse milte-julte 5 chunks chahiye — poore chunks nahi."

Official naam: Isko "Similarity Search" ya "Retrieval" kehte hain (ye "R" hai RAG ke naam mein). 5 ka matlab top-5 — sabse zyada similar 5 chunks.

Kaise kaam karta hai: state.prompt (user ka sawaal) bhi embedding se ek vector banta hai, fir Qdrant us vector ke sabse paas wale 5 stored-vectors dhoondta hai (jaise "sabse paas ke GPS coordinates").


### Step 7 — Chunks Ko Ek Text Mein Jodna

const context = relevantDocs.map((d) => d.pageContent).join("\n\n");
Soch: "5 chunks mile hain (5 alag objects), lekin LLM ko ek hi clean text chahiye — mujhe inhe jodना hai."

Official naam: Isko "Context Building" kehte hain — ye "context" wahi hai jo LLM ko "extra knowledge" ki tarah diya jaayega.

\n\n (double newline) se jodne ka reason: har chunk ke beech clear separation rahe, LLM confuse na ho ki kaha ek chunk khatam hua.


### Step 8 — Prompt Banana (Augmentation)

const messages = [
  new SystemMessage(`You are CortexAI PDF Assistant.
Rules:
- Answer ONLY from the uploaded PDF.
- Never make up information.
- If the answer is not present in the PDF, reply: "..."
- Use Markdown formatting.`),
  new HumanMessage(`Context:${context}\nQuestion:${state.prompt}`),
];
Soch: "Ab mujhe context + sawaal, dono ko LLM ko ek saath dena hai, aur LLM ko clearly bolna hai ki 'sirf isi context se jawab de, khud se mat bana'."

Official naam: Isko "Prompt Augmentation" kehte hain — ye RAG ka "A" (Augmented) hai. SystemMessage mein jo rule hai "Answer ONLY from the uploaded PDF... Never make up information" — isko "Grounding" kehte hain (LLM ko "grounded" rakhna, matlab wo apni marzi se facts invent na kare — isse "hallucination" rokte hain).


### Step 9 — LLM Se Jawab Lena

const llm = await getModel("pdfRag");
const response = await llm.invoke(messages);
Soch: "Ab sab kuch ready hai, LLM ko bhejo aur jawab lo."

Official naam: Isko "Generation" kehte hain — RAG ka "G". Ye normal LLM call hai, bas isko diya gaya context "grounded" hai.


### Step 10 — Credits Kaatna + Response Dena

await deductCredits(state.userId, "pdf");
return { ...state, aiResponse: response.content };
Soch: "Jawab successfully mil gaya, ab user ka credit kaato (business logic), aur result wapas bhejo."

💡 Yaad rakhne wali cheez: Credits response milne ke baad kaate — agar pehle katते, aur LLM fail ho jaata, user ko bina result ke charge ho jaata.


### Step 11 — Cleanup

finally {
  try {
    fs.unlinkSync(state.file.path);
  } catch (err) {
    console.log("Error deleting file:", err);
  }
}
Soch: "Chahe sab sahi hua ho ya error aaya ho, uploaded temp PDF ko delete karna hai — disk space bachana hai."


## 4. Isko Kisी Aur Ko Kaise Samjhao (Elevator Pitch)

30-second version (interview ya casual poochhe to):

"Maine ek RAG-based PDF assistant banaya hai. Jab user PDF upload karta hai, main uska text nikaal ke chunks mein todता hoon, har chunk ko embeddings se vector mein convert karke Qdrant (vector database) mein store karता hoon. Jab user sawaal poochhta hai, us sawaal se sabse relevant chunks dhoondta hoon (similarity search se), aur sirf wahi chunks LLM ko context ki tarah deta hoon jawab banane ke liye — poori PDF nahi. Isse LLM sirf उपलब्ध document se jawab deta hai, khud se kuch bana ke nahi bolta."


#### 2-minute version (technical interview ke liye): Upar wala + ye add karo:


"Chunking ke liye RecursiveCharacterTextSplitter use kiya, 1000-character chunks with 200-character overlap — taaki sentence boundary pe context na tute. Embedding Gemini ke model se banayi, aur Qdrant mein top-5 similarity-search karta hoon per query. Prompt mein explicit rule daali hai ki LLM sirf diye gaye context se jawab de, hallucinate na kare — isko grounding kehte hain."


## 5. Terminology Glossary — "Isko Kya Kehte Hain"

| Tumhare code mein kya | Iska official naam | Kya karta hai
 |
| pdf.getText() | Text Extraction | PDF se raw text nikalna
 |
| RecursiveCharacterTextSplitter | Chunking / Text Splitting | Text ko chhote pieces mein todna
 |
| docs (chunks ka array) | Documents / Chunks | Har chhota text-piece
 |
| chunkOverlap | Overlap | Chunks ke beech repeated text, context na tute isliye
 |
| embeddings | Embedding Model | Text ko vector (numbers) mein badalna
 |
| vector (numbers ki list) | Vector / Embedding | Text ka numeric representation
 |
| QdrantVectorStore | Vector Database / Vector Store | Vectors ko store + search karne wali database
 |
| collectionName | Collection | Database ke andar ek "table" jaisa grouping
 |
| similaritySearch() | Retrieval / Similarity Search | Sabse relevant chunks dhoondna
 |
| context variable | Retrieved Context | Mile hue relevant chunks, jode hue
 |
| System + Human message jodna | Prompt Augmentation | Context + sawaal ko LLM-ready banana
 |
| "Answer ONLY from PDF" rule | Grounding | LLM ko facts invent karne se rokna
 |
| llm.invoke() | Generation | LLM se final jawab lena
 |
| Poora process | RAG Pipeline | Sab steps mila ke
 |

## 6. Interview Questions

🟢 Easy

#### Q1. RAG ka full form kya hai aur ye kya karta hai?


Retrieval-Augmented Generation. Ye pehle relevant information dhoondta hai (Retrieval), usse prompt mein add karta hai (Augmented), fir LLM se jawab banata hai (Generation) — taaki LLM sirf diye gaye data se jawab de, khud se na bana le.


#### Q2. Chunking kyun zaroori hai?


Kyunki LLM ek baar mein limited text hi process kar sakta hai (context limit), aur poori badi file dena mehenga aur inaccurate hota hai. Chunking se hum sirf relevant chhota hissa LLM ko dete hain.


#### Q3. Embedding kya hota hai, simple bhasha mein?


Text ko numbers ki ek list (vector) mein badalna, jisse similar-meaning texts ke numbers bhi similar (paas-paas) hote hain — isse computer "similarity" samajh pata hai.


#### Q4. Vector database normal database (jaise MongoDB) se kaise alag hai?


Normal database exact-match ya condition-based search karta hai (jaise userId: 123). Vector database "similarity"-based search karta hai — "ye naya vector, stored vectors mein se sabse kiske paas hai" — matlab meaning ke aadhar pe search, exact text-match pe nahi.

🟡 Medium

#### Q5. chunkOverlap kya hai aur ye kyun zaroori hai?


Chunks ke beech kuch text repeat hota hai (jaise 200 characters) taaki agar koi important sentence chunk-boundary pe tut raha ho, wo dono chunks mein poori tarah maujood rahe — context loss na ho.


#### Q6. similaritySearch(prompt, 5) mein "5" ka kya matlab hai, aur agar hum "20" karein to kya farak padega?


"5" matlab top-5 sabse relevant chunks liye jaa rahe hain. Agar "20" karein, zyada context milega (potentially better accuracy), lekin prompt bada ho jaayega (zyada cost, LLM ke liye zyada "noise" bhi ho sakta hai agar irrelevant chunks bhi aa jaayen). Ye ek trade-off hai — accuracy vs cost/speed.


#### Q7. Agar user PDF mein jo nahi likha hai wo poochhe, to system kya karega?


System prompt mein explicit rule hai: "If the answer is not present in the PDF, reply: 'I couldn't find this information...'" — isse LLM ko bataya gaya hai ki agar retrieved context mein jawab na ho, to khud se bana ke mat bolo (hallucinate mat karo), seedha "nahi mila" bol do.


#### Q8. fromDocuments aur fromExistingCollection mein kya farak hai?


fromDocuments naya collection banata hai aur naye documents/chunks ko embed karke usme insert karta hai — pehli baar data daalne ke liye. fromExistingCollection sirf ek pehle-se-bani collection se connect karta hai, sirf query/search karne ke liye — naya data insert nahi karta.

🔴 Hard

#### Q9. Agar ek hi PDF baar-baar upload ho (same content), to kya har baar naye embeddings banana zaroori hai? Isko optimize kaise karoge?


Nahi, agar file same hai, dobara embed karna wasteful hai. Optimize karne ke liye: file ka hash (jaise MD5/SHA) nikaal ke check karo ki wo hash pehle se kisi collection mein exist karta hai ya nahi — agar haan, purani collection reuse karo, naya nahi banao. Isse compute aur cost dono bachte hain.


#### Q10. Chunking ke 2 tarike hote hain — fixed-size vs semantic chunking. Tumne kaunsa use kiya aur dusra kyun better/worse ho sakta hai?


Maine RecursiveCharacterTextSplitter use kiya, jo character-count based hai (fixed size, jaise 1000 chars) — ye fast aur simple hai, lekin kabhi-kabhi ek sentence ya paragraph beech mein bhi kaat sakta hai. Semantic chunking iski jagah paragraph/topic-boundaries pe todता hai (meaning-aware), jo zyada accurate hota hai lekin slower/complex hai — production-grade systems mein semantic chunking better result deता hai, khaaskar structured documents (legal, medical) ke liye.


#### Q11. Tumhara system PDF ke content ke saath-saath "table" ya "chart" jaisi cheezein handle kar payega? Agar nahi, kya limitation hai?


pdf-parse mostly plain text extract karta hai — tables ka structure (rows/columns) ya charts ka visual data usually flatten/lost ho jaata hai (sirf text jaisa dikh sakta hai, bina proper structure ke). Isko better banane ke liye specialized PDF-table-extraction libraries (jaise pdf-table-extractor, ya vision-model se PDF-pages ko image ki tarah bhi process kar sakte hain) chahiye honge.


#### Q12. Agar 2 alag PDFs ka context ek dusre se contradict kare (jaise dono mein alag numbers hon), to system kya karega? Isko kaise handle karoge?


Abhi ka system har PDF ke liye alag collection banata hai, isliye ek query sirf uसी PDF ke andar search karti hai — cross-PDF contradiction ka scene hi nahi banta (ek time pe ek hi PDF ka context use hota hai). Agar future mein multiple PDFs ek saath query karne ho (jaise ek shared knowledge-base), to contradiction handle karne ke liye: (1) retrieved chunks mein source (kaunsi PDF se aaya) tag karo, (2) LLM ko bolo agar conflicting info mile to dono sources mention kare, user ko khud decide karne do.


## 7. Future Mein Kya Aur Seekhna Chahiye (RAG Advanced Topics)

Halka-halka overview — jab time mile, ek-ek karke explore karna:

| Topic | Kya hai (ek line mein)
 |
| Semantic Chunking | Character-count ki jagah meaning/paragraph ke hisaab se chunk karna — zyada accurate
 |
| Hybrid Search | Sirf vector-similarity nahi, keyword-search (jaise BM25) bhi mix karna — kuch cases mein exact-word match zaroori hota hai jo pure-vector search miss kar deta hai
 |
| Re-ranking | Top-20 chunks nikaal ke, ek dusre (chhote, sasta) model se unko re-order karna taaki sabse best 5 hi LLM ko jaaye — accuracy improve karta hai
 |
| Metadata Filtering | Chunks ke saath extra info (jaise "page number", "date", "author") store karna, taaki search ke waqt filter bhi laga sako ("sirf 2024 ke documents mein dhoondo")
 |
| Multi-query Retrieval | User ke ek sawaal ko LLM se 3-4 alag tarike se rephrase karwana, sabse search karna, results combine karna — better recall
 |
| Streaming Responses | LLM ka jawab word-by-word turant dikhana (jaise ChatGPT), poora wait na karwana
 |
| Caching | Same/similar queries ke liye pehle se compute kiye jawab reuse karna — cost aur speed dono behtar
 |
| Evaluation (RAGAS) | RAG system "kitna accurate" hai, ye measure karne ke liye specialized tools/metrics (faithfulness, relevance score)
 |
| Multi-modal RAG | Sirf text nahi, images/tables/charts ko bhi properly samajh ke retrieve karna
 |
Sabse pehle seekhne layak (priority order): Metadata Filtering → Hybrid Search → Re-ranking — ye teen sabse jaldi "real-world usable" impact dete hain.


## 8. 🧠 Yaad Rakhne Ki Tricks (Mnemonics)


### Trick 1 — Khud "RAG" naam hi order bata raha hai!

Isko overthink mat karo — naam mein hi answer hai:

R-A-G
R = Retrieval    (pehle dhoondo)
A = Augmented    (fir jodo)
G = Generation   (aakhir mein LLM se jawab)
Jab bhi confuse ho "pehle kya, baad mein kya" — bas RAG spell karo, order mil jaayega.


### Trick 2 — 5 Official Stages: "Load Shedding Se Raat Gayi"

Industry (LangChain docs) mein RAG ke 5 official stages hote hain: Load → Split → Store → Retrieve → Generate. Ek Hinglish sentence bana lo jisse first-letters match karein:

"Load Shedding Se Raat Gayi" Load → Shedding(Split) → Se(Store) → Raat(Retrieve) → Gayi(Generate)

| Sentence word | Stage | Tumhare code mein
 |
| Load | Text extract karna | pdf.getText()
 |
| Shedding (Split) | Chunking | RecursiveCharacterTextSplitter
 |
| Se (Store) | Embed + Vector-DB mein save | vectorStore()
 |
| Raat (Retrieve) | Similarity search | similaritySearch()
 |
| Gayi (Generate) | LLM se jawab | llm.invoke()
 |
Power-cut wala fun fact yaad rakhoge, poori RAG-pipeline order bhi saath mein yaad aa jaayegi.


### Trick 3 — Ek Chhoti Si Kahani (Memory-Palace Style)

Ye poore pipeline ko ek kahani ki tarah yaad karo — kahaniyan facts se zyada der tak yaad rehti hain:

Ek Postman (Extraction) PDF se ek chitthi (text) leke aata hai. Chitthi bahut lambi hai, isliye use chhoti-chhoti parchiyon (Chunks) mein phaada jaata hai — thoda kinara overlap rakhke, taaki koi baat beech mein na tut jaaye. Har parchi ko ek unique address (Embedding/Vector) mil jaata hai. Saari parchiyan ek godown (Vector Store / Qdrant) mein rakh di jaati hain, address ke hisaab se sorted. Jab koi sawaal aata hai, ek jasoos (Retriever) godown mein jaake sabse milte-julte address wali 5 parchiyan dhoondh laata hai. Ek assistant (Augmentation) un 5 parchiyon ko jod ke ek clean note banata hai. Aakhir mein ek genie (LLM/Generator) us note ko padhta hai aur jawab deta hai — genie ko saaf bola gaya hai: "sirf isi note se jawab dena, khud se kahani mat banana" (Grounding).

Agli baar jab RAG bhool jaao, bas ye kahani yaad karo — Postman → Parchiyan → Address → Godown → Jasoos → Assistant → Genie.


### Trick 4 — Chunk Size aur Overlap ka Ratio

Exact numbers (1000, 200) yaad rakhne ki jagah, ratio yaad rakho:

"Chunk ka 20% wapas do agle ko"

1000 ka 20% = 200 — kitna bhi chunkSize rakho (500 ho ya 2000), overlap uska ~15-20% rakhna hai. Number bhool jaoge to bhi ratio se nikal loge.


### Trick 5 — fromDocuments vs fromExistingCollection

D se Documents  = D se Daalna     → naya data INSERT karna hai
E se Existing   = E se Ek baar bana hua, ab sirf padhna hai
"Daalna hai to Documents, sirf dekhna hai to Existing"


### Trick 6 — await Bhoolne Se Bachne Ki Habit

Ek chhota sawaal khud se poocho har naye function-call pe:

"Iska naam 'async' function hai kya? Haan? To await lagao — warna Promise haath mein aayega, result nahi."

Chhoti si body-trick: jab bhi code likho aur koi .something() call kar rahe ho jiska result turant use karna hai — ek second ruk ke us function ki definition dekh lo. Agar async likha mile, await bina socho mat chhodो.


### Trick 7 — Grounding Yaad Rakhne Ka One-Liner

"Genie sirf bottle ke andar se jawab dega, bahar ki duniya usko nahi pata"

Matlab: LLM ko explicitly bolna hai "sirf diye gaye context se jawab do" — warna wo apni general-knowledge se "hallucinate" (bana ke bol) sakta hai.


## 9. ⚡ 30-Second Revision

RAG = Retrieval (dhoondo) + Augmented (jodo) + Generation (LLM se jawab)
Chunking → text ko chhote pieces mein todna (overlap = context na tute)
Embedding → text ko "numbers" (vector) mein badalna, similar-meaning = paas-paas numbers
Vector DB (Qdrant) → un vectors ko store + similarity-search karna
fromDocuments (naya insert) vs fromExistingCollection (sirf purane se connect)
Grounding → LLM ko bolna "sirf diye gaye context se jawab do, mat banao"
Credits/cleanup hamesha response milne ke baad, finally block mein
Ye notes pdfRag.agent.js (Kaido project) ke real code se banaye gaye hain — jab bhi RAG bhool jaao, ye file se shuru se revise kar lena.
