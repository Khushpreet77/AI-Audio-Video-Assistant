# 🎙️ AI Audio Video Assistant

An AI-powered **Audio/Video Meeting Assistant** that transforms YouTube videos and local audio/video files into searchable, structured knowledge.

The application transcribes media, generates summaries and meeting insights, and uses **Retrieval-Augmented Generation (RAG)** to let users ask questions about the content and receive answers grounded in the generated transcript.

> **⚠️ Learning Project & Attribution**
>
> This project is **not my original idea**.
>
> I built this project by learning from a YouTube tutorial created by an experienced AI professional and implementing the concepts myself.
>
> The purpose of this repository is to demonstrate my **hands-on understanding and implementation of AI and RAG concepts**, rather than to claim ownership of the original project idea.
>
> I intentionally give credit to the original creator for the concept and learning material.
>
> This project represents an important stage of my learning journey: **studying professional implementations, understanding how the components work together, implementing them myself, debugging the system, and gradually building the foundation needed to create original AI applications.**

---

## 📌 About the Project

**AI Audio Video Assistant** processes audio/video content and converts it into an interactive knowledge source.

Users can provide:

* A **YouTube URL**
* A **local audio file**
* A **local video file**

The application then processes the media through an AI pipeline:

```text
Audio / Video / YouTube URL
            │
            ▼
     Audio Acquisition
       & Conversion
            │
            ▼
        Audio Chunking
            │
            ▼
     Speech-to-Text
   ┌────────┴────────┐
   │                 │
 Whisper          Sarvam AI
 English          Hinglish
   │                 │
   └────────┬────────┘
            ▼
        Transcript
            │
      ┌─────┼──────────────┐
      ▼     ▼              ▼
   Summary  Extraction   RAG Pipeline
            │              │
            │              ▼
            │        Text Chunking
            │              │
            │              ▼
            │         Embeddings
            │              │
            │              ▼
            │        Chroma Vector DB
            │              │
            │              ▼
            │         Similarity Search
            │              │
            │              ▼
            │        Relevant Context
            │              │
            │              ▼
            │          Mistral LLM
            │              │
            └──────────────┤
                           ▼
                    User's Answer
```

The main pipeline is orchestrated in `main.py`, while the Streamlit application provides the interactive interface in `app.py`.

---

# ✨ Features

### 🎬 Multi-Source Input

The application accepts:

* YouTube URLs through `yt-dlp`
* Local audio files
* Local video files

The input is converted into a standardized WAV-based processing pipeline.

### 🎙️ Speech-to-Text

The project supports two transcription paths:

* **Whisper** for English transcription
* **Sarvam AI** for Hinglish transcription with English translation

Audio is processed in chunks so that longer media can be handled efficiently.

### 📝 Automatic Summarization

The transcript is split into manageable sections, each section is summarized, and the partial summaries are combined into a final professional summary using Mistral.

### 🔍 Meeting Intelligence

The application extracts:

* Action items
* Responsible owners
* Deadlines when mentioned
* Key decisions
* Unresolved/open questions

These are generated from the transcript using dedicated LLM chains.

### 🧠 RAG-Based Question Answering

The core AI feature is the RAG pipeline.

The transcript is:

1. Split into smaller chunks
2. Converted into embeddings
3. Stored in Chroma
4. Retrieved using similarity search
5. Passed as context to the Mistral LLM

The LLM is explicitly instructed to answer **only from the retrieved transcript context**, reducing the chance of answers unrelated to the source material.

### 💬 Interactive RAG Chat

After processing the media, users can ask questions such as:

```text
"What were the main decisions?"

"Who was responsible for the deployment task?"

"What problems were discussed?"

"What did they say about RAG?"

"Were there any unresolved questions?"
```

The question is passed through the retriever and the relevant transcript context is supplied to the LLM before generating the answer.

---

# 🧠 RAG Architecture

The RAG implementation is the most important learning component of this project.

```text
                Transcript
                    │
                    ▼
          RecursiveCharacterTextSplitter
                    │
                    ▼
              Text Chunks
                    │
                    ▼
        all-MiniLM-L6-v2 Embeddings
                    │
                    ▼
              Chroma Vector DB
                    │
                    ▼
            Similarity Retriever
                 top-k = 4
                    │
                    ▼
           Relevant Transcript
                Context
                    │
                    ▼
             Mistral LLM
                    │
                    ▼
                 Answer
```

The vector store uses **HuggingFace's `all-MiniLM-L6-v2` embedding model** and Chroma for persistent local vector storage. The retriever uses similarity search and retrieves the top four relevant chunks by default.

The RAG prompt also contains an important grounding rule:

> Answer based only on the meeting transcript context.

If the information cannot be found in the retrieved context, the system is instructed to say that it could not find the information in the transcript.

---

# 📂 Project Structure

```text
AI-Audio-Video-Assistant/
│
├── core/
│   ├── extractor.py
│   ├── rag_engine.py
│   ├── summarizer.py
│   ├── transcriber.py
│   └── vector_store.py
│
├── utils/
│   └── audio_processor.py
│
├── vector_db/
│   └── chroma.sqlite3
│
├── downloads/
│
├── app.py
├── main.py
├── requirement.txt
└── .gitignore
```

---

# 📁 File-by-File Explanation

## `app.py`

This is the **Streamlit application layer**.

It provides the interface through which the user:

* Enters a YouTube URL or local file path
* Selects the language
* Starts media analysis
* Views the generated title
* Reads the summary
* Views the transcript
* Views extracted action items
* Views key decisions
* Views open questions
* Chats with the RAG assistant

The Streamlit app calls the underlying processing modules rather than implementing the AI logic itself.

---

## `main.py`

This is the **CLI/application pipeline entry point**.

It orchestrates the complete workflow:

```text
process_input()
      ↓
transcribe_all()
      ↓
generate_title()
      ↓
summarize()
      ↓
extract_action_items()
      ↓
extract_key_decisions()
      ↓
extract_questions()
      ↓
build_rag_chain()
```

It also provides a terminal-based chat loop where users can ask questions about the processed content.

---

# 📁 `core/`

The `core` directory contains the main AI processing components.

---

## `core/audio_processor.py`

**Actually located under `utils/`, not `core/`.**

This module handles media preparation.

It includes functions for:

### YouTube audio download

Uses `yt-dlp` to retrieve the best available audio from a YouTube URL.

### Audio conversion

Uses Pydub/FFmpeg to convert local audio/video files into WAV.

### Audio normalization

The converted audio is standardized to:

```text
Mono
16 kHz
WAV
```

### Audio chunking

Long audio is divided into smaller chunks before transcription.

### Input routing

`process_input()` determines whether the supplied source is a URL or local file and sends it through the appropriate processing path.

---

## `core/transcriber.py`

Responsible for **Speech-to-Text**.

The module supports:

### Whisper

For English:

```text
Audio
 ↓
Local Whisper Model
 ↓
Transcript
```

The Whisper model is loaded lazily and defaults to the `small` model.

### Sarvam AI

For Hinglish:

```text
Hinglish Audio
      ↓
Sarvam Speech-to-Text
      ↓
English Transcript
```

Because the Sarvam synchronous endpoint has a duration limit, longer chunks are further divided into 25-second pieces before being sent to the API.

---

## `core/summarizer.py`

Responsible for **LLM-based summarization and title generation**.

The transcript is split using:

```text
chunk_size = 3000
chunk_overlap = 200
```

Each chunk is summarized individually and the partial summaries are then combined into a final meeting summary.

It also generates a short professional title based on the transcript.

---

## `core/extractor.py`

Responsible for extracting structured information from the transcript.

It contains separate LLM chains for:

```text
Transcript
    │
    ├──→ Action Items
    │
    ├──→ Key Decisions
    │
    └──→ Open Questions
```

For action items, the system attempts to identify:

* Task description
* Owner
* Deadline

This module demonstrates how an LLM can be used not only for free-form generation but also for extracting useful structured insights from unstructured text.

---

## `core/vector_store.py`

This module implements the **vector database layer of the RAG system**.

It:

1. Splits the transcript
2. Creates `Document` objects
3. Generates embeddings
4. Stores the vectors in Chroma
5. Creates a similarity-based retriever

The project uses:

```text
Embedding Model:
all-MiniLM-L6-v2

Vector Store:
Chroma

Chunk Size:
500

Chunk Overlap:
50

Retriever:
Similarity Search

Top-K:
4
```

The Chroma database is persisted locally in the `vector_db/` directory.

---

## `core/rag_engine.py`

This is the **central RAG module**.

It connects:

```text
Retriever
    +
Prompt
    +
Mistral LLM
    ↓
RAG Chain
```

The `build_rag_chain()` function:

1. Builds the vector store
2. Creates the retriever
3. Initializes the Mistral LLM
4. Defines the grounding prompt
5. Connects everything using LangChain LCEL

The core LCEL flow is conceptually:

```text
                    User Question
                          │
              ┌───────────┴───────────┐
              ▼                       ▼
         Retriever               Question
              │
              ▼
       Relevant Documents
              │
              ▼
        Format Context
              │
              └───────────┬───────────┘
                          ▼
                       Prompt
                          ▼
                    Mistral LLM
                          ▼
                       Answer
```

This is the main component demonstrating my understanding and implementation of **Retrieval-Augmented Generation**.

---

# 🛠️ Tech Stack

| Technology         | Purpose                                 |
| ------------------ | --------------------------------------- |
| Python             | Core application                        |
| Streamlit          | User interface                          |
| yt-dlp             | YouTube audio acquisition               |
| Pydub              | Audio processing and conversion         |
| FFmpeg             | Media/audio processing                  |
| OpenAI Whisper     | Local English speech-to-text            |
| Sarvam AI          | Hinglish speech-to-text and translation |
| LangChain          | LLM orchestration and RAG pipeline      |
| Mistral AI         | LLM for generation and analysis         |
| HuggingFace        | Text embeddings                         |
| `all-MiniLM-L6-v2` | Embedding model                         |
| ChromaDB           | Persistent vector database              |

These dependencies are reflected in the project's `requirement.txt`.

---

# 📚 What I Learned

The main purpose of this project was to move beyond simply learning individual AI concepts and understand how they work together in a **complete AI application**.

### RAG

I learned how to build an end-to-end RAG pipeline involving:

* Document preparation
* Text chunking
* Embeddings
* Vector databases
* Similarity search
* Retrievers
* Context construction
* Prompting
* LLM generation

### Embeddings

I learned how textual information can be converted into numerical vector representations that allow semantic similarity search.

### Vector Databases

I gained hands-on experience using **ChromaDB** to persist embeddings and retrieve relevant transcript chunks.

### LangChain LCEL

I learned how multiple components can be composed into a pipeline using LangChain's runnable/LCEL approach.

### Grounded Generation

I learned the importance of providing retrieved context to an LLM and explicitly instructing it to answer from that context rather than relying only on its general knowledge.

### Speech-to-Text Pipeline

I learned how raw audio/video can be transformed into text that can subsequently become the knowledge source for an RAG system.

---

# 🔄 End-to-End Workflow

The complete application can be understood as:

```text
                 USER INPUT
                     │
          ┌──────────┴──────────┐
          │                     │
     YouTube URL            Local File
          │                     │
       yt-dlp              Pydub/FFmpeg
          │                     │
          └──────────┬──────────┘
                     ▼
                 WAV Audio
                     │
                     ▼
                Audio Chunks
                     │
                     ▼
              Speech-to-Text
             ┌───────┴───────┐
             │               │
          Whisper         Sarvam
          English        Hinglish
             │               │
             └───────┬───────┘
                     ▼
                 Transcript
                     │
        ┌────────────┼─────────────┐
        ▼            ▼             ▼
     Summary     Extraction       RAG
        │            │             │
        │            │       Embeddings
        │            │             │
        │            │        ChromaDB
        │            │             │
        │            │        Retriever
        │            │             │
        │            │             ▼
        │            │       Relevant Context
        │            │             │
        └────────────┴─────────────┤
                                   ▼
                              Mistral LLM
                                   │
                                   ▼
                              User Answer
```

---

# 🚀 Getting Started

## 1. Clone the repository

```bash
git clone https://github.com/Khushpreet77/AI-Audio-Video-Assistant.git
cd AI-Audio-Video-Assistant
```

## 2. Create a virtual environment

```bash
python -m venv venv
```

Activate it on Windows:

```bash
venv\Scripts\activate
```

## 3. Install dependencies

```bash
pip install -r requirement.txt
```

The project also requires the **FFmpeg binary** to be installed separately for audio/video processing.

## 4. Configure environment variables

Create a `.env` file and add the required API credentials, including the Mistral and, when using Hinglish transcription, Sarvam credentials.

Example:

```env
MISTRAL_API_KEY=your_mistral_api_key
SARVAM_API_KEY=your_sarvam_api_key
```

The application loads environment variables using `python-dotenv`.

## 5. Run the Streamlit application

```bash
streamlit run app.py
```

Then provide a YouTube URL or local media file and start the analysis.

---

# 🖥️ Usage

### Example Input

```text
https://www.youtube.com/watch?v=...
```

or a local file:

```text
C:/videos/meeting.mp4
```

Select the language:

```text
English
```

or:

```text
Hinglish
```

Then run the analysis.

The application produces:

```text
✓ Transcript
✓ Generated title
✓ Summary
✓ Action items
✓ Key decisions
✓ Open questions
✓ RAG-powered chat
```

---

# 🎯 Project Learning Objective

This repository is primarily a **learning and implementation project**.

I built it to understand how modern AI applications combine multiple components into one pipeline:

**Media Processing → Speech-to-Text → LLM Processing → Embeddings → Vector Search → RAG → Answer Generation**

The most important part of this project for me was not simply making the application work, but understanding **why each component exists and how information flows from the original media to the final answer**.

---

# 🙏 Credits & Attribution

The **original project idea and learning material were taken from a YouTube tutorial by an experienced AI professional**.

I want to explicitly acknowledge that:

**I am not claiming the original idea or concept as my own.**

This repository represents my own implementation and hands-on learning while following the tutorial.

I created and published this repository to document my progress in learning:

* Generative AI
* RAG
* LangChain
* Vector databases
* Embeddings
* Speech-to-text
* LLM application development

I believe that learning from professionals by studying their implementations and rebuilding projects is an important step toward eventually creating original solutions.

---

# 🔮 Future Improvements

Possible future improvements include:

* Support for more audio/video formats
* Better handling of long-form media
* Timestamp-aware retrieval
* Source citations for retrieved transcript chunks
* Multiple-document knowledge bases
* Conversation memory across sessions
* Improved multilingual support
* More efficient vector-store management
* Video-frame understanding for truly multimodal question answering

---

## 👩‍💻 Author

**Khushpreet**

B.Tech CSE — Data Science

Currently learning and building with:

**Python · Data Science · Machine Learning · Generative AI · RAG · LangChain**

> **Learn from professionals → Understand the concepts → Implement them yourself → Experiment → Build something original.**
