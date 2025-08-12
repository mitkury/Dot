### Overview

This app lets you chat with your documents entirely locally. You pick a folder, the app parses supported files, chunks them, and turns those chunks into embeddings. It stores a local FAISS index. When you ask a question, the app retrieves the most relevant chunks from that index, builds a prompt with that context, and uses a local LLM to generate an answer. The UI streams back both the sources (for example, PDF pages) and the answer text.

- Select a folder to ingest; supported types include PDF, DOCX, PPTX, Markdown, and many code/text formats.
- Files are split into overlapping chunks and embedded with a lightweight transformer model.
- A FAISS vector index is saved under your Documents folder for fast local retrieval.
- Asking a question performs top-k semantic retrieval, prompts a local LLM with that context, and streams the answer.
- Source documents are surfaced alongside the answer; PDFs are shown inline with page anchors.

The sections below detail the indexing pipeline, the retrieval-augmented chat flow, and where to configure behavior.

### Attachments analysis/indexing and Doc Chat flow

This document explains how documents (“attachments”) are ingested, analyzed, and indexed, and how “Chat with Docs” (RAG) works end-to-end in this project. It references concrete files, functions, and line numbers.

## Indexing pipeline (attachments → FAISS vector store)

- **User trigger (Upload documents)**
  - UI button lives in `src/index.html` at lines 44–52.
  - Renderer wires the button to open a directory chooser in `src/render.js` at lines 641–643, invoking `open-dialog`.
  - Main process shows the directory dialog and kicks off indexing in `src/index.js` at lines 388–399, which calls `processSelectedDirectory(...)`.

- **Main process orchestration**
  - `processSelectedDirectory(selectedPath, webContents)` (progress + IPC updates) in `src/index.js` lines 163–182. It delegates to the actual indexer function `processDirectory` and streams progress to the renderer via `update-progress`.

```163:182:/workspace/src/index.js
async function processSelectedDirectory(selectedPath, webContents) {
    const progressBar = new cliProgress.SingleBar({}, cliProgress.Presets.shades_classic);
    progressBar.start(100, 0);

    const updateProgress = (progress) => {
        progressBar.update(progress);
        webContents.send('update-progress', progress);
    };

    try {
        await processDirectory(selectedPath, updateProgress);
        progressBar.stop();
        webContents.send('loading-complete', selectedPath);
    } catch (error) {
        console.error('Error processing directory:', error);
        progressBar.stop();
        webContents.send('loading-error', error.message);
    }
}
```

- **Indexer (build embeddings + FAISS store)**
  - The main indexer is `processDirectory(directory, updateProgress)` in `aadotllm/embeddings.js` lines 29–127.
  - It reads chunking config (`readConfig`) at lines 19–27, sets defaults (33–39), builds an embeddings model (50–52), loads supported files with a `DirectoryLoader` (54–87), splits into chunks (37–40, 97–99), batches into FAISS (100–117), emits progress (119–121), and saves the FAISS index to `~/Documents/Dot-Data` (42, 123–124).

```29:60:/workspace/aadotllm/embeddings.js
const processDirectory = async (directory, updateProgress) => {
  const { FaissStore, RecursiveCharacterTextSplitter, PDFLoader, DocxLoader, PPTXLoader, NotionLoader, TextLoader, DirectoryLoader, HuggingFaceTransformersEmbeddings } = await importESModules();

  const configPath = path.join(__dirname, 'config.json');
  const config = await readConfig(configPath);
  const chunkSize = config.chunk_length || 4000;
  const chunkOverlap = config.chunk_overlap || 2000;

  const textSplitter = new RecursiveCharacterTextSplitter({
    chunkSize,
    chunkOverlap,
  });

  const desktopPath = path.join(os.homedir(), "Documents", "Dot-Data");
  await fs.mkdir(desktopPath, { recursive: true });

  const embeddings = new HuggingFaceTransformersEmbeddings({
    modelName: "Xenova/all-MiniLM-L6-v2",
  });

  const loader = new DirectoryLoader(
    directory,
    {
      ".pdf": (path) => new PDFLoader(path),
      ".docx": (path) => new DocxLoader(path),
      ".pptx": (path) => new PPTXLoader(path),
      ".md": (path) => new NotionLoader(path),
      ".js": (path) => new TextLoader(path),
      ".py": (path) => new TextLoader(path),
      ".ts": (path) => new TextLoader(path),
      ".mjs": (path) => new TextLoader(path),
      ".c": (path) => new TextLoader(path),
      ".cpp": (path) => new TextLoader(path),
      ".cs": (path) => new TextLoader(path),
      ".java": (path) => new TextLoader(path),
      ".go": (path) => new TextLoader(path),
      ".rb": (path) => new TextLoader(path),
      ".swift": (path) => new TextLoader(path),
      ".kt": (path) => new TextLoader(path),
      ".php": (path) => new TextLoader(path),
      ".html": (path) => new TextLoader(path),
      ".css": (path) => new TextLoader(path),
      ".xml": (path) => new TextLoader(path),
      ".json": (path) => new TextLoader(path),
      ".yaml": (path) => new TextLoader(path),
      ".yml": (path) => new TextLoader(path),
      ".r": (path) => new TextLoader(path),
      ".sh": (path) => new TextLoader(path),
      ".bat": (path) => new TextLoader(path),
      ".pl": (path) => new TextLoader(path),
      ".rs": (path) => new TextLoader(path),
      ".scala": (path) => new TextLoader(path),
      ".sql": (path) => new TextLoader(path),
    },
    { recursive: true,
      ignoreFiles: (filePath) => {
        const ext = path.extname(filePath).toLowerCase();
        return !this[ext];
      }
    }
  );
```

```97:127:/workspace/aadotllm/embeddings.js
  const docs = await loader.load();
  const Documentato = await textSplitter.splitDocuments(docs);

  const batchSize = 10;
  let batch = [];
  let Vittorio;

  for (let i = 0; i < Documentato.length; i++) {
    batch.push(Documentato[i]);

    if (batch.length === batchSize || i === Documentato.length - 1) {
      const tempStore = await FaissStore.fromDocuments(batch, embeddings);

      if (Vittorio) {
        Vittorio.mergeFrom(tempStore);
      } else {
        Vittorio = tempStore;
      }

      batch = [];
    }

    const progress = ((i + 1) / Documentato.length) * 100;
    updateProgress(progress);
  }

  await Vittorio.save(desktopPath);
  console.log({ Documentato });
};
```

- **Renderer progress UI**
  - Receives progress in `src/render.js` lines 645–654, updates the spinner percentage and stroke offset.

## Chat with Docs (RAG)

- **Module selection (Doc Dot vs Big Dot)**
  - Dropdown UI in `src/index.html` lines 89–144.
  - Renderer toggles the active module by sending `switch-script` IPC in `src/render.js` lines 692–701.
  - Main process switches the active chat module in `src/index.js` lines 212–223. Default is Doc Dot on startup (685–701).

- **User sends a message**
  - Renderer click handler builds a new bot message container and invokes `run-chat` in `src/render.js` lines 454–480.
  - Main process handles `run-chat` and calls the active chat module’s `runChat(input, sendToken, configPath)` in `src/index.js` lines 188–206.

```188:206:/workspace/src/index.js
ipcMain.handle('run-chat', async (event, userInput) => {
    console.log(`IPC call received to run ${currentScript}`);
    try {
        const sendToken = (token) => {
            console.log('Sending token:', token);
            event.sender.send('chat-token', token);
        };
        if (activeChatModule) {
            const response = await activeChatModule(userInput, sendToken, configPath);
            console.log('Final response:', response);
            return response;
        } else {
            throw new Error(`Module ${currentScript} is not loaded.`);
        }
    } catch (error) {
        console.error(`Error running ${currentScript}:`, error);
        throw error;
    }
});
```

- **Doc chat implementation (retrieval + generation)**
  - `runChat(input, sendToken, configPath)` in `aadotllm/docdot.js` orchestrates RAG:
    - Reads config (21–42) to get `max_tokens`, `n_ctx`, `sources`, and `ggufFilePath`.
    - Instantiates local LLM `LlamaCpp` (76–83) and embeddings model (86–89).
    - Loads FAISS from `~/Documents/Dot-Data` (91–94) and builds a retriever for top-`sources` (96).
    - Prompt template and chain assembly (98–120) combining retriever context with the LLM.
    - Streams output (125–131), emits source context first (155–161), renders PDF sources as iframes with page anchors (170–189), then streams the model’s answer tokens with stop conditions (121–123, 195–227). Returns the final concatenated response (236–240).

```90:120:/workspace/aadotllm/docdot.js
  // Load vector store from a specified directory
  const directory = path.join(os.homedir(), "Documents", "Dot-Data");

  const VectorStore = await FaissStore.load(directory, embeddings);

  // Equivalent to kwargs in python, 2 indicates the 2 closest documents will be provided
  const retriever = VectorStore.asRetriever(sources, {});

  const prompt = PromptTemplate.fromTemplate(`
  You are an assistant for question-answering tasks. Use the following pieces of retrieved context to answer the question. If you don't know the answer, just say that you don't know. Keep the answer concise.
  Question: {input}
  
  Context: {context}
  
  Answer:
  `);

  const ragChainFromDocs = RunnableSequence.from([
    RunnablePassthrough.assign({
      context: (input) => formatDocumentsAsString(input.context),
    }),
    prompt,
    llm,
    new StringOutputParser(),
  ]);

  let ragChainWithSource = new RunnableMap({
    steps: { context: retriever, input: new RunnablePassthrough() },
  });
  ragChainWithSource = ragChainWithSource.assign({ answer: ragChainFromDocs });
```

```155:189:/workspace/aadotllm/docdot.js
        // Check if the key is 'context' to start processing tokens
        if (key === 'context') {
          processTokens = true;
          // Send "Source" token before processing context
          sendToken("Source:");
        }

        if (processTokens) {
          // ... existing code ...

          // Process and format the context into iframe HTML if the source is a PDF
          if (key === 'context') {
            let context;
            try {
              // Assuming chunk[key] is already a valid JSON string
              context = chunk[key];

              const sources = context.map(doc => {
                const sourcePath = doc.metadata.source;
                const pageNumber = doc.metadata.loc?.pageNumber; // Use optional chaining to safely access pageNumber
                const fileExtension = sourcePath.split('.').pop().toLowerCase();

                if (fileExtension === 'pdf') {
                  const sourcePathWithPage = pageNumber ? `${sourcePath}#page=${pageNumber}` : sourcePath;
                  return `<iframe src="${sourcePathWithPage}" style="width:100%; height:300px; border: 1px solid #ccc; box-shadow: 0 4px 6px rgba(0,0,0,0.1); margin: 10px 0;" frameborder="0"></iframe>`;
                }
                return `<p>${sourcePath}</p>`;
              }).join('\n');

              sendToken(sources);
            } catch (error) {
              console.error("Error parsing context:", error);
              throw error;
            }
          } else {
            // ... existing code ...
```

- **Renderer streaming + rendering**
  - Receives tokens via `chat-token` and appends them to the last bot message in `src/render.js` lines 171–192.
  - Splits out `<iframe>` tokens and injects them once at the beginning of the message; the remaining text tokens are rendered as Markdown and typeset with MathJax in `src/render.js` lines 398–450.

```171:192:/workspace/src/render.js
ipcRenderer.on('chat-token', (event, token) => {
    console.log('Received token:', token);
    accumulatedTokens += token; // Append the new token to the accumulated tokens
    appendTokenToLastMessage();

    // Clear any existing timeout
    if (tokenStreamTimeoutId) {
        clearTimeout(tokenStreamTimeoutId);
    }

    // Set a new timeout
    tokenStreamTimeoutId = setTimeout(() => {
        messageStreamingComplete = true;
        console.log('Message streaming complete.');
        if (autoTtsEnabled) {
            const lastMessage = document.querySelector('.bot-bubble .text-container')?.innerText || '';
            console.log('Auto TTS processing last message:', lastMessage);
            sendMessageToMainForTTS(lastMessage, () => resetSpeakerIcon(speakerIcon), handleTtsError);
        }
    }, TOKEN_STREAM_TIMEOUT);
});
```

```398:450:/workspace/src/render.js
function appendTokenToLastMessage() {
    const chatContainer = document.getElementById('bot-message');
    const lastMessage = chatContainer.lastElementChild;
    if (lastMessage && lastMessage.classList.contains('message')) {
        const botBubble = lastMessage.querySelector('.bot-bubble');
        let textContainer = lastMessage.querySelector('.text-container'); // A container for text content

        if (!textContainer) {
            // Create a container for text content if it doesn't exist
            textContainer = document.createElement('div');
            textContainer.classList.add('text-container');
            botBubble.appendChild(textContainer);
        }

        if (botBubble) {
            // Separate iframes from other tokens
            let newContent = '';
            const tokens = accumulatedTokens.split('</iframe>'); // Split tokens by iframe end tag

            tokens.forEach(token => {
                if (token.includes('<iframe')) {
                    if (!iframesInserted) {
                        iframes += token + '</iframe>'; // Include the iframe end tag
                    }
                } else {
                    newContent += token;
                }
            });

            // Append iframes only once
            if (!iframesInserted && iframes) {
                botBubble.insertAdjacentHTML('afterbegin', iframes); // Insert iframes at the beginning
                iframesInserted = true; // Mark iframes as inserted
            }

            // Append new content to the text container
            textContainer.innerHTML = marked(newContent);

            // Log the content of the textContainer
            console.log("Updated botBubble content:", textContainer.innerHTML);

            // Check if MathJax is loaded and then typeset
            if (window.MathJax) {
                MathJax.typesetPromise([textContainer]).then(() => {
                    console.log("MathJax has finished processing!");
                }).catch((err) => console.error('MathJax processing error:', err));
            } else {
                console.log("MathJax is not available to process the content.");
            }
        }
    }
}
```

## Configuration and storage

- **User config path**: main process sets `configPath` to Electron `userData` in `src/index.js` lines 7–9. If missing, a default config is created at lines 827–839 (includes `n_ctx`, `n_batch`, `max_tokens`, `sources`, etc.). This path is passed into chat modules as the third argument.
- **Doc index location**: both the indexer and the doc chat module use `~/Documents/Dot-Data`:
  - Save FAISS: `aadotllm/embeddings.js` lines 42 and 123–124.
  - Load FAISS: `aadotllm/docdot.js` lines 91–94.
- **Key knobs:**
  - Chunking: `aadotllm/embeddings.js` lines 33–39.
  - Retrieval top-k: `sources` in config, used in `aadotllm/docdot.js` line 96.
  - Context size and max tokens: `aadotllm/docdot.js` lines 66–83 and stop/stream control at 121–123, 195–227.

## Big Dot (non-RAG chat)

- For general chat without retrieval, `aadotllm/bigdot.js` provides `runChat(input, sendToken, configPath)` that builds a `ChatLlamaCpp` chain without document retrieval. Selection is controlled via the same `switch-script` IPC (`src/index.js` 212–223; renderer `src/render.js` 692–701; UI `src/index.html` 89–144).

## End-to-end sequence (Doc Dot)

- User clicks “Upload documents” → directory chooser opens → `processSelectedDirectory` runs → `processDirectory` builds embeddings and saves FAISS index to `~/Documents/Dot-Data` → renderer shows progress and populates the file tree.
- User types a question → renderer invokes `run-chat` → main calls `aadotllm/docdot.js/runChat` → loads FAISS, retrieves top-k chunks, formats a prompt, streams tokens back → renderer inserts iframes for sources and renders the streamed Markdown answer.
