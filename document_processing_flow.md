# Document Processing Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                          RAW DOCUMENT FILE                          │
│                         (course1_script.txt)                        │
├─────────────────────────────────────────────────────────────────────┤
│ Course Title: Introduction to Python                                │
│ Course Link: https://example.com/python                             │
│ Course Instructor: Jane Doe                                         │
│                                                                      │
│ Lesson 1: Variables and Data Types                                  │
│ Lesson Link: https://example.com/python/lesson1                     │
│ Variables are containers for storing data values. Python has        │
│ several data types including strings, integers, and floats...       │
│                                                                      │
│ Lesson 2: Control Flow                                              │
│ Lesson Link: https://example.com/python/lesson2                     │
│ Control flow statements allow you to execute code conditionally...  │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│              STEP 1: FILE READING (read_file)                       │
│              • Read with UTF-8 encoding                             │
│              • Fallback to error-ignoring mode if needed            │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│         STEP 2: METADATA EXTRACTION (lines 110-146)                 │
│         Parse first 3-4 lines using regex                           │
├─────────────────────────────────────────────────────────────────────┤
│  Extract:                                                           │
│  • Course Title: "Introduction to Python"                           │
│  • Course Link: "https://example.com/python"                        │
│  • Instructor: "Jane Doe"                                           │
│                                                                      │
│  Create Course Object:                                              │
│  ┌─────────────────────────────────────────┐                       │
│  │ Course {                                │                       │
│  │   title: "Introduction to Python"       │                       │
│  │   course_link: "https://..."            │                       │
│  │   instructor: "Jane Doe"                │                       │
│  │   lessons: []                           │                       │
│  │ }                                       │                       │
│  └─────────────────────────────────────────┘                       │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│          STEP 3: LESSON PARSING (lines 162-243)                     │
│          Scan for "Lesson N: Title" markers                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Lesson 1 Found:                                                    │
│  ┌─────────────────────────────────────────┐                       │
│  │ Lesson {                                │                       │
│  │   lesson_number: 1                      │                       │
│  │   title: "Variables and Data Types"     │                       │
│  │   lesson_link: "https://..."            │                       │
│  │ }                                       │                       │
│  │                                         │                       │
│  │ Content: "Variables are containers..."  │                       │
│  │          [~5000 characters]             │                       │
│  └─────────────────────────────────────────┘                       │
│                      │                                              │
│                      ▼                                              │
│  Lesson 2 Found:                                                    │
│  ┌─────────────────────────────────────────┐                       │
│  │ Lesson {                                │                       │
│  │   lesson_number: 2                      │                       │
│  │   title: "Control Flow"                 │                       │
│  │   lesson_link: "https://..."            │                       │
│  │ }                                       │                       │
│  │                                         │                       │
│  │ Content: "Control flow statements..."   │                       │
│  │          [~4000 characters]             │                       │
│  └─────────────────────────────────────────┘                       │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│           STEP 4: TEXT CHUNKING (chunk_text)                        │
│           For each lesson's content (lines 25-91)                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  A. Sentence Splitting (regex)                                      │
│     "Variables are containers. Python supports many types."         │
│     ↓                                                               │
│     ["Variables are containers.", "Python supports many types."]    │
│                                                                      │
│  B. Build chunks (800 char max, sentence boundaries)                │
│                                                                      │
│     ┌────────────────────────┐                                      │
│     │ Chunk 0 (800 chars)    │                                      │
│     │ "Variables are..."     │                                      │
│     │ [15 sentences]         │                                      │
│     └────────────────────────┘                                      │
│              │                                                       │
│              │ ◄─── 100 char overlap                                │
│              ▼                                                       │
│     ┌────────────────────────┐                                      │
│     │ Chunk 1 (800 chars)    │                                      │
│     │ "...Python supports.." │                                      │
│     │ [14 sentences]         │                                      │
│     └────────────────────────┘                                      │
│              │                                                       │
│              │ ◄─── 100 char overlap                                │
│              ▼                                                       │
│     ┌────────────────────────┐                                      │
│     │ Chunk 2 (800 chars)    │                                      │
│     │ "...data types..."     │                                      │
│     └────────────────────────┘                                      │
│                                                                      │
│  Config: chunk_size=800, chunk_overlap=100                          │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│        STEP 5: CONTEXT ENHANCEMENT (lines 184-188, 234)             │
│        Add course + lesson context to each chunk                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Original chunk:                                                    │
│  "Variables are containers for storing data values..."              │
│                                                                      │
│  Enhanced with context:                                             │
│  "Course Introduction to Python Lesson 1 content: Variables are..." │
│                                                                      │
│  Why? Helps embeddings understand where content comes from!         │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│       STEP 6: CREATE CourseChunk OBJECTS (lines 190-197)            │
│       Wrap each chunk with metadata                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────────────────────────────────────────────┐          │
│  │ CourseChunk {                                        │          │
│  │   content: "Course Introduction to Python Lesson..." │          │
│  │   course_title: "Introduction to Python"             │          │
│  │   lesson_number: 1                                   │          │
│  │   chunk_index: 0                                     │          │
│  │ }                                                    │          │
│  └──────────────────────────────────────────────────────┘          │
│  ┌──────────────────────────────────────────────────────┐          │
│  │ CourseChunk {                                        │          │
│  │   content: "Course Introduction to Python Lesson..." │          │
│  │   course_title: "Introduction to Python"             │          │
│  │   lesson_number: 1                                   │          │
│  │   chunk_index: 1                                     │          │
│  │ }                                                    │          │
│  └──────────────────────────────────────────────────────┘          │
│  ┌──────────────────────────────────────────────────────┐          │
│  │ CourseChunk {                                        │          │
│  │   content: "Course Introduction to Python Lesson..." │          │
│  │   course_title: "Introduction to Python"             │          │
│  │   lesson_number: 2                                   │          │
│  │   chunk_index: 2                                     │          │
│  │ }                                                    │          │
│  └──────────────────────────────────────────────────────┘          │
│                                                                      │
│  Returns: (Course, List[CourseChunk])                               │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│         STEP 7: EMBEDDING & STORAGE (rag_system.py)                 │
│         Convert to vectors and store in ChromaDB                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  For each CourseChunk:                                              │
│                                                                      │
│  1. Generate embedding using Sentence Transformer                   │
│     "Course Intro... Lesson 1 content: Variables..."                │
│                    ↓                                                │
│     [0.042, -0.183, 0.271, ..., 0.094]  ← 384-dim vector           │
│                                                                      │
│  2. Store in ChromaDB "course_content" collection                   │
│     ┌───────────────────────────────────────┐                      │
│     │ ChromaDB Entry:                       │                      │
│     │ • id: "chunk_0"                       │                      │
│     │ • embedding: [vector]                 │                      │
│     │ • document: [text]                    │                      │
│     │ • metadata: {                         │                      │
│     │     course_title: "Intro to Python"   │                      │
│     │     lesson_number: 1                  │                      │
│     │     chunk_index: 0                    │                      │
│     │   }                                   │                      │
│     └───────────────────────────────────────┘                      │
│                                                                      │
│  Now searchable with semantic similarity!                           │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    READY FOR QUERIES! 🎉                            │
│                                                                      │
│  User: "What are variables?"                                        │
│     ↓                                                               │
│  Embed query → Search ChromaDB → Find relevant chunks              │
│     ↓                                                               │
│  Returns chunks from Lesson 1 with high similarity                  │
│     ↓                                                               │
│  Claude generates answer using retrieved context                    │
└─────────────────────────────────────────────────────────────────────┘


## Key Features

✓ **Sentence-Aware**: Never breaks mid-sentence
✓ **Overlap**: 100 chars overlap prevents context loss at boundaries
✓ **Context-Enhanced**: Each chunk knows its course + lesson
✓ **Metadata-Rich**: Can filter by course or lesson number
✓ **Semantic Search Ready**: Embeddings enable conceptual matching
```
