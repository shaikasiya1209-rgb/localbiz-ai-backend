# LocalBiz AI

A beginner-friendly local Python API that answers small-business questions from uploaded TXT, Markdown, CSV, and text-based PDF documents. FastAPI provides the test page at `http://127.0.0.1:8000/docs`; there is no separate frontend.

## Project structure

The project folder is `C:\Users\SK.Ayesha\localbiz-ai`. Open this folder in VS Code. The working files are:

```text
localbiz-ai/
|-- app.py
|-- requirements.txt
|-- README.md
|-- sample_data/
|   `-- business_catalog.txt
|-- .venv/                 (pre-existing Python environment)
|-- .venv312/              (Python 3.12 environment used for this project)
`-- chroma_data/           (created automatically when the app runs)
```

`app.py` extracts document text, splits it into chunks, persists those chunks in ChromaDB, finds matching passages, and asks Ollama/Gemma to answer from those passages. Retrieval uses simple local word-feature vectors rather than downloading a separate embedding model. It is a prototype, so check the returned sources and answers before relying on them.

## 1. Check Python and open the project

Install Python 3.11 or 3.12 and VS Code if needed. ChromaDB's binary dependencies may not yet support the newest Python releases, so use Python 3.11 or 3.12 for this project.

In VS Code, choose **File > Open Folder** and open `C:\Users\SK.Ayesha\localbiz-ai`. Choose **Terminal > New Terminal** and verify the terminal is in the project folder:

```powershell
Get-Location
```

The path should end with `localbiz-ai`. If it does not:

```powershell
cd C:\Users\SK.Ayesha\localbiz-ai
```

To see installed Python versions, run `py -0p`.

## 2. Create and activate the virtual environment

Run these commands from the project folder. Check an existing environment before reusing it:

```powershell
\.venv\Scripts\python.exe --version
```

If it reports Python 3.11 or 3.12, activate it. If `.venv` does not exist, create it with Python 3.12:

```powershell
py -3.12 -m venv .venv
.\.venv\Scripts\Activate.ps1
```

If your existing `.venv` uses another Python version, leave it in place and make a separate Python 3.12 environment:

```powershell
py -3.12 -m venv .venv312
.\.venv312\Scripts\Activate.ps1
```

If you have Python 3.11 instead, replace `-3.12` with `-3.11`. When activated, the prompt starts with the environment name, such as `(.venv)` or `(.venv312)`. If PowerShell says script execution is disabled, allow activation for this terminal window only, then activate again:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
.\.venv\Scripts\Activate.ps1
```

You can also run the environment's Python directly as `\.venv\Scripts\python.exe` without activating it.

## 3. Install Python packages

`requirements.txt` is in the project root. Install its contents:

```powershell
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

## 4. Install and start Ollama

1. Download and install Ollama for Windows from [ollama.com/download](https://ollama.com/download).
2. Open a new PowerShell terminal and verify the install:

   ```powershell
   ollama --version
   ```

3. Download Gemma (this may take a while):

   ```powershell
   ollama pull gemma3:1b
   ```

4. Ollama normally runs in the background after installation. If it is not running, open a separate terminal and run `ollama serve`; leave that terminal open.
5. Optional model check:

   ```powershell
   ollama run gemma3:1b
   ```

   Ask it a question, then type `/bye`. Keep Ollama available while using the API.

## 5. Run the backend

In the project terminal, activate the environment you installed the packages into (`.venv312` in this workspace), then start the server:

```powershell
python -m uvicorn app:app --reload
```

Wait for `Uvicorn running on http://127.0.0.1:8000`. Keep this terminal open. Open [http://127.0.0.1:8000/docs](http://127.0.0.1:8000/docs) in a browser. Swagger lets you try every endpoint without frontend code.

## 6. Test the endpoints

### `GET /` - welcome

Open `http://127.0.0.1:8000/` or use `GET /` in Swagger. Expected response:

```json
{
  "message": "Welcome to LocalBiz AI",
  "docs": "/docs",
  "health": "/health"
}
```

### `GET /health` - API and Ollama status

This checks whether Ollama is reachable and whether the model is available. Example when everything is ready:

```json
{
  "status": "ok",
  "api": "ok",
  "ollama": {
    "available": true,
    "model": "gemma3:1b",
    "model_available": true,
    "message": "Ollama and the requested model are ready."
  },
  "stored_chunks": 0
}
```

A `degraded` status means the API is working but Ollama or the model needs attention. Read the `message` field for the next step.

### `POST /upload` - upload the sample catalog

In `/docs`, expand **POST /upload**, click **Try it out**, choose `sample_data/business_catalog.txt` for `file`, and click **Execute**. Expected response:

```json
{
  "message": "Document uploaded successfully!",
  "filename": "business_catalog.txt",
  "chunks_stored": 1
}
```

TXT, MD, CSV, and PDF are supported. Scanned/image-only PDFs are rejected because this project does not include OCR.

### `GET /documents` - list stored documents

After uploading the catalog, use this endpoint. Example:

```json
{
  "count": 1,
  "documents": [
    {
      "filename": "business_catalog.txt",
      "chunks": 1,
      "uploaded_at": "2026-10-16T12:00:00+00:00"
    }
  ]
}
```

The timestamp will be your actual upload time.

### `POST /ask` - ask a question

In `/docs`, expand **POST /ask**, click **Try it out**, enter one of the JSON requests below, and click **Execute**:

```json
{"question": "What is the return policy for damaged items?"}
```

```json
{"question": "Which products cost less than INR 500?"}
```

```json
{"question": "How long does delivery take?"}
```

```json
{"question": "Do you sell laptops?"}
```

Gemma's exact wording can vary. The response includes an answer plus source filenames and matching text, for example:

```json
{
  "answer": "Damaged or incorrect items should be reported within 48 hours with photos.",
  "sources": [
    {
      "filename": "business_catalog.txt",
      "text": "...Damaged or incorrect items should be reported within 48 hours with photos..."
    }
  ]
}
```

When no matching passage is found, the API returns `The uploaded documents do not specify this information.` with an empty `sources` list and does not call Ollama.

## Endpoints at a glance

| Method | Path | Purpose |
| --- | --- | --- |
| GET | `/` | Welcome message and links |
| GET | `/health` | API, Ollama, and model status |
| POST | `/upload` | Upload a TXT, MD, CSV, or PDF file |
| GET | `/documents` | List filenames and chunk counts |
| POST | `/ask` | Retrieve passages and generate an answer |
| GET | `/docs` | Interactive Swagger testing page |

## Troubleshooting

- **PowerShell cannot find `.venv\Scripts\Activate.ps1`:** Run `Get-Location`. Change to the project directory with `cd C:\Users\SK.Ayesha\localbiz-ai`, then run `\.venv\Scripts\Activate.ps1`. The environment is inside the project folder.
- **`python` or `py` is not recognized:** Install Python from python.org, enable **Add Python to PATH**, reopen VS Code, and check with `py -0p`.
- **Dependency installation fails on a very new Python:** Install Python 3.11 or 3.12. Preserve the existing environment and create `.venv312` with `py -3.12 -m venv .venv312`, activate it, and repeat the install step.
- **PowerShell blocks activation:** Use the process-only execution policy command in Step 2; it applies only to the current terminal window.
- **`ollama` is not recognized:** Install Ollama, open a new PowerShell terminal, then check `ollama --version`.
- **Health reports Ollama unavailable:** Start Ollama. If it did not start automatically, run `ollama serve` in a second terminal and retry `GET /health`.
- **Health reports the model missing:** Run `ollama pull gemma3:1b` and retry.
- **`/ask` says no documents are uploaded:** First upload a file with `POST /upload`, then check `GET /documents`.
- **Upload rejects the file type:** Use a `.txt`, `.md`, `.csv`, or `.pdf` file.
- **PDF has no extractable text:** It may be a scan. This version reads text-based PDFs; use a text-based PDF or OCR the scan before uploading.
- **Port 8000 is busy:** Stop the other server with Ctrl+C, or run `python -m uvicorn app:app --reload --port 8001` and open `http://127.0.0.1:8001/docs`.

## 2.5-hour hackathon demo

1. Set up the project and Python environment: about 25 minutes.
2. Install packages and Ollama, then download Gemma: about 30 minutes (download time varies).
3. Start the API and upload the sample catalog: about 15 minutes.
4. Demo returns, delivery, prices, and the unknown-laptop question: about 20 minutes.
5. Rehearse the explanation and reserve time for issues: about 20 minutes.

**Problem:** Small businesses spend time answering repeat questions about products and policies. **Solution:** Upload business documents and ask questions against them. **Technology:** Python, FastAPI, ChromaDB, and Ollama with Gemma. **Future scope:** Improve multilingual retrieval, add stock updates, and build a customer-facing interface.
