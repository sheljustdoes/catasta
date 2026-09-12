# catasta

Modular FastAPI + Next.js scaffold for serving ML research projects as polished, interactive demos.

catasta is a reusable project scaffold for turning research pipelines into
production-quality interactive demos. It pairs a FastAPI backend (where your
science lives) with a Next.js + TypeScript frontend (where people experience
it). Designed to be forked per project — swap the pipeline, adjust the UI,
deploy. The architecture stays the same; the research changes.

---

## Architecture

```
project-demo/
├── api/                        # FastAPI backend (Python)
│   ├── main.py                 # App entrypoint + CORS config
│   ├── routes/
│   │   └── predict.py          # Prediction endpoint(s)
│   ├── core/
│   │   ├── config.py           # Env vars, model paths, settings
│   │   └── models.py           # Pydantic request/response schemas
│   ├── services/
│   │   └── pipeline.py         # Your ML pipeline — the actual science
│   ├── utils/
│   │   └── preprocessing.py    # Input transforms, validation, formatting
│   ├── requirements.txt
│   └── Dockerfile
│
├── web/                        # Next.js frontend (TypeScript)
│   ├── app/
│   │   ├── layout.tsx          # Root layout, fonts, metadata
│   │   ├── page.tsx            # Landing / demo page
│   │   └── results/
│   │       └── page.tsx        # Results display (if multi-page)
│   ├── components/
│   │   ├── ui/                 # Reusable primitives (button, card, etc.)
│   │   ├── upload/
│   │   │   └── Dropzone.tsx    # File upload component
│   │   ├── results/
│   │   │   └── ResultCard.tsx  # Individual result display
│   │   └── layout/
│   │       ├── Header.tsx
│   │       └── Footer.tsx
│   ├── lib/
│   │   ├── api.ts              # API client — calls to FastAPI
│   │   └── types.ts            # Shared TypeScript types
│   ├── public/
│   │   └── og-image.png        # Social preview image
│   ├── tailwind.config.ts
│   ├── next.config.ts
│   └── package.json
│
├── notebooks/                  # Optional — exploration, validation
│   └── demo_walkthrough.ipynb
│
└── README.md
```

---

## API — FastAPI Backend

### `api/main.py`

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from routes.predict import router as predict_router
from core.config import settings

app = FastAPI(
    title=settings.PROJECT_NAME,
    description=settings.PROJECT_DESCRIPTION,
    version=settings.VERSION,
)

app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.ALLOWED_ORIGINS,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

app.include_router(predict_router, prefix="/api/v1")


@app.get("/health")
async def health():
    return {"status": "ok", "project": settings.PROJECT_NAME}
```

### `api/core/config.py`

```python
from pydantic_settings import BaseSettings
from typing import List


class Settings(BaseSettings):
    # ---- Swap these per project ----
    PROJECT_NAME: str = "iridis"
    PROJECT_DESCRIPTION: str = "Skin tone phenotype classification from imaging data"
    VERSION: str = "0.1.0"

    # Model / pipeline config
    MODEL_PATH: str = "./models/weights.pt"
    DEVICE: str = "cpu"

    # Server
    ALLOWED_ORIGINS: List[str] = [
        "http://localhost:3000",       # local Next.js dev
        "https://your-app.vercel.app", # production frontend
    ]

    class Config:
        env_file = ".env"


settings = Settings()
```

### `api/core/models.py`

```python
from pydantic import BaseModel
from typing import List, Optional, Dict, Any


# ---- Adapt these schemas per project ----

class PredictionRequest(BaseModel):
    """For non-file inputs (text, params, JSON payloads)."""
    input_data: Dict[str, Any]
    options: Optional[Dict[str, Any]] = None


class PredictionResult(BaseModel):
    """Single result item — a classification, a score, a cluster, etc."""
    label: str
    confidence: Optional[float] = None
    metadata: Optional[Dict[str, Any]] = None


class PredictionResponse(BaseModel):
    """Wrapper for one or more results."""
    project: str
    results: List[PredictionResult]
    input_summary: Optional[str] = None
    processing_time_ms: Optional[float] = None
```

### `api/routes/predict.py`

```python
import time
from fastapi import APIRouter, UploadFile, File, HTTPException
from core.config import settings
from core.models import PredictionResponse, PredictionResult
from services.pipeline import Pipeline

router = APIRouter()

# Initialize pipeline once at startup
pipeline = Pipeline(model_path=settings.MODEL_PATH, device=settings.DEVICE)


@router.post("/predict", response_model=PredictionResponse)
async def predict(file: UploadFile = File(...)):
    """
    Primary prediction endpoint.
    Accepts file upload — swap to JSON body for non-file inputs.
    """
    if not file.content_type.startswith("image/"):
        raise HTTPException(status_code=400, detail="Expected an image file")

    start = time.time()

    contents = await file.read()
    results = pipeline.run(contents)
    elapsed = (time.time() - start) * 1000

    return PredictionResponse(
        project=settings.PROJECT_NAME,
        results=results,
        input_summary=f"{file.filename} ({file.content_type})",
        processing_time_ms=round(elapsed, 2),
    )
```

### `api/services/pipeline.py`

```python
from typing import List
from core.models import PredictionResult


class Pipeline:
    """
    Your ML pipeline lives here. This is the only file that changes
    significantly between projects.

    Pattern:
      1. __init__  — load model, load artifacts, warm up
      2. preprocess — raw input → model-ready tensor/array
      3. predict   — model inference
      4. postprocess — raw output → structured PredictionResults
      5. run       — orchestrates the above
    """

    def __init__(self, model_path: str, device: str = "cpu"):
        self.device = device
        self.model = self._load_model(model_path)

    def _load_model(self, path: str):
        """Load your trained model/weights/artifacts."""
        # Example:
        # import torch
        # model = torch.load(path, map_location=self.device)
        # model.eval()
        # return model
        pass

    def preprocess(self, raw_input: bytes):
        """
        Transform raw input into model-ready format.
        For images: decode, resize, normalize, to tensor.
        For text: tokenize, encode.
        For tabular: validate schema, transform features.
        """
        pass

    def predict(self, processed_input):
        """Run inference. Return raw model output."""
        pass

    def postprocess(self, raw_output) -> List[PredictionResult]:
        """
        Convert raw model output into structured results.
        This is where you map indices to labels, apply thresholds,
        format confidence scores, attach metadata.
        """
        # Example:
        # return [
        #     PredictionResult(
        #         label="tone_cluster_14",
        #         confidence=0.92,
        #         metadata={"lab_values": [65.2, 12.1, 8.4]}
        #     )
        # ]
        pass

    def run(self, raw_input: bytes) -> List[PredictionResult]:
        """Full pipeline: preprocess → predict → postprocess."""
        processed = self.preprocess(raw_input)
        raw_output = self.predict(processed)
        return self.postprocess(raw_output)
```

---

## Frontend — Next.js + TypeScript + Tailwind

### `web/lib/types.ts`

```typescript
// Mirror your API response schemas

export interface PredictionResult {
  label: string;
  confidence?: number;
  metadata?: Record<string, any>;
}

export interface PredictionResponse {
  project: string;
  results: PredictionResult[];
  input_summary?: string;
  processing_time_ms?: number;
}

// App-level state
export interface DemoState {
  status: "idle" | "uploading" | "processing" | "done" | "error";
  file: File | null;
  preview: string | null;
  response: PredictionResponse | null;
  error: string | null;
}
```

### `web/lib/api.ts`

```typescript
import { PredictionResponse } from "./types";

const API_BASE = process.env.NEXT_PUBLIC_API_URL || "http://localhost:8000";

export async function predict(file: File): Promise<PredictionResponse> {
  const formData = new FormData();
  formData.append("file", file);

  const res = await fetch(`${API_BASE}/api/v1/predict`, {
    method: "POST",
    body: formData,
  });

  if (!res.ok) {
    const error = await res.json().catch(() => ({ detail: "Request failed" }));
    throw new Error(error.detail || `API error: ${res.status}`);
  }

  return res.json();
}
```

### `web/app/layout.tsx`

```tsx
import type { Metadata } from "next";
import { Inter } from "next/font/google";
import "./globals.css";

// ---- Swap font per project for distinct personality ----
const font = Inter({
  subsets: ["latin"],
  variable: "--font-sans",
});

export const metadata: Metadata = {
  // ---- Swap per project ----
  title: "iridis — Perceptual Skin Tone Classification",
  description:
    "Explore 40+ computationally recovered skin tone phenotypes from 2M+ diverse images.",
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en" className={font.variable}>
      <body className="min-h-screen bg-neutral-50 text-neutral-900 antialiased">
        {children}
      </body>
    </html>
  );
}
```

### `web/app/page.tsx`

```tsx
"use client";

import { useState, useCallback } from "react";
import { Dropzone } from "@/components/upload/Dropzone";
import { ResultCard } from "@/components/results/ResultCard";
import { predict } from "@/lib/api";
import type { DemoState } from "@/lib/types";

export default function DemoPage() {
  const [state, setState] = useState<DemoState>({
    status: "idle",
    file: null,
    preview: null,
    response: null,
    error: null,
  });

  const handleFile = useCallback(async (file: File) => {
    const preview = URL.createObjectURL(file);
    setState({ status: "processing", file, preview, response: null, error: null });

    try {
      const response = await predict(file);
      setState((prev) => ({ ...prev, status: "done", response }));
    } catch (err) {
      setState((prev) => ({
        ...prev,
        status: "error",
        error: err instanceof Error ? err.message : "Something went wrong",
      }));
    }
  }, []);

  const reset = () => {
    if (state.preview) URL.revokeObjectURL(state.preview);
    setState({ status: "idle", file: null, preview: null, response: null, error: null });
  };

  return (
    <main className="mx-auto max-w-3xl px-6 py-16">
      {/* ---- Project header — swap per project ---- */}
      <header className="mb-12">
        <h1 className="text-3xl font-semibold tracking-tight">iridis</h1>
        <p className="mt-2 text-neutral-500">
          Perceptual skin tone classification from imaging data
        </p>
      </header>

      {/* Upload or results */}
      {state.status === "idle" && <Dropzone onFile={handleFile} />}

      {state.status === "processing" && (
        <div className="flex flex-col items-center gap-4 py-12">
          {state.preview && (
            <img
              src={state.preview}
              alt="Uploaded"
              className="h-48 w-48 rounded-lg object-cover"
            />
          )}
          <p className="text-sm text-neutral-400">Processing…</p>
        </div>
      )}

      {state.status === "done" && state.response && (
        <div className="space-y-6">
          <div className="flex items-center justify-between">
            <p className="text-sm text-neutral-400">
              {state.response.processing_time_ms}ms
            </p>
            <button
              onClick={reset}
              className="text-sm text-neutral-500 underline underline-offset-2 hover:text-neutral-800"
            >
              Try another
            </button>
          </div>

          <div className="flex gap-6">
            {state.preview && (
              <img
                src={state.preview}
                alt="Input"
                className="h-48 w-48 shrink-0 rounded-lg object-cover"
              />
            )}
            <div className="space-y-3">
              {state.response.results.map((result, i) => (
                <ResultCard key={i} result={result} />
              ))}
            </div>
          </div>
        </div>
      )}

      {state.status === "error" && (
        <div className="space-y-4 py-12 text-center">
          <p className="text-red-600">{state.error}</p>
          <button
            onClick={reset}
            className="text-sm text-neutral-500 underline underline-offset-2"
          >
            Try again
          </button>
        </div>
      )}
    </main>
  );
}
```

### `web/components/upload/Dropzone.tsx`

```tsx
"use client";

import { useCallback, useState, useRef } from "react";

interface DropzoneProps {
  onFile: (file: File) => void;
  accept?: string;       // MIME types, e.g. "image/*"
  maxSizeMB?: number;
}

export function Dropzone({
  onFile,
  accept = "image/*",
  maxSizeMB = 10,
}: DropzoneProps) {
  const [dragging, setDragging] = useState(false);
  const inputRef = useRef<HTMLInputElement>(null);

  const handleDrop = useCallback(
    (e: React.DragEvent) => {
      e.preventDefault();
      setDragging(false);
      const file = e.dataTransfer.files[0];
      if (file) onFile(file);
    },
    [onFile]
  );

  const handleChange = useCallback(
    (e: React.ChangeEvent<HTMLInputElement>) => {
      const file = e.target.files?.[0];
      if (file) onFile(file);
    },
    [onFile]
  );

  return (
    <div
      role="button"
      tabIndex={0}
      onDragOver={(e) => {
        e.preventDefault();
        setDragging(true);
      }}
      onDragLeave={() => setDragging(false)}
      onDrop={handleDrop}
      onClick={() => inputRef.current?.click()}
      onKeyDown={(e) => e.key === "Enter" && inputRef.current?.click()}
      className={`
        flex cursor-pointer flex-col items-center justify-center
        rounded-xl border-2 border-dashed px-6 py-16
        transition-colors
        ${
          dragging
            ? "border-neutral-900 bg-neutral-100"
            : "border-neutral-300 hover:border-neutral-400"
        }
      `}
    >
      <p className="text-sm text-neutral-500">
        Drop a file here, or click to browse
      </p>
      <p className="mt-1 text-xs text-neutral-400">
        Up to {maxSizeMB}MB
      </p>
      <input
        ref={inputRef}
        type="file"
        accept={accept}
        onChange={handleChange}
        className="hidden"
      />
    </div>
  );
}
```

### `web/components/results/ResultCard.tsx`

```tsx
import type { PredictionResult } from "@/lib/types";

interface ResultCardProps {
  result: PredictionResult;
}

export function ResultCard({ result }: ResultCardProps) {
  return (
    <div className="rounded-lg border border-neutral-200 bg-white p-4">
      <div className="flex items-baseline justify-between">
        <span className="font-medium">{result.label}</span>
        {result.confidence !== undefined && (
          <span className="text-sm text-neutral-400">
            {(result.confidence * 100).toFixed(1)}%
          </span>
        )}
      </div>

      {result.metadata && (
        <div className="mt-2 space-y-1">
          {Object.entries(result.metadata).map(([key, value]) => (
            <div key={key} className="flex justify-between text-sm">
              <span className="text-neutral-500">{key}</span>
              <span className="font-mono text-neutral-700">
                {typeof value === "number" ? value.toFixed(2) : String(value)}
              </span>
            </div>
          ))}
        </div>
      )}
    </div>
  );
}
```

---

## Adapting Per Project

The scaffold is designed so that swapping projects requires changes in only a few places:

### Backend — what to change:

| File | What to swap |
|------|-------------|
| `core/config.py` | `PROJECT_NAME`, `PROJECT_DESCRIPTION`, `MODEL_PATH` |
| `core/models.py` | Request/response schemas to match your pipeline's I/O |
| `services/pipeline.py` | The actual science — load, preprocess, predict, postprocess |
| `routes/predict.py` | Input type (`UploadFile` for images, `Body` for JSON, etc.) |

### Frontend — what to change:

| File | What to swap |
|------|-------------|
| `app/layout.tsx` | Title, description, font choice |
| `app/page.tsx` | Header text, input component (Dropzone vs text input vs form) |
| `components/results/ResultCard.tsx` | Result display (cards, charts, visualizations) |
| `lib/types.ts` | Type definitions to match new API response shape |
| `lib/api.ts` | Endpoint path if changed; input encoding if not FormData |

### Input patterns per project type:

| Project | Input | Component |
|---------|-------|-----------|
| **iridis** | Image upload | `Dropzone` (image/*) |
| **lambent** | Image upload | `Dropzone` (image/*) |
| **topos** | Genomic data file (.csv, .vcf) | `Dropzone` (accept=".csv,.vcf") |
| **veridian** | Search query (text) | Text input + submit |
| **recolo** | Conversation / text | Text area + submit |

---

## Deployment

### Backend → Railway (or Render)
```bash
# From api/ directory
railway init
railway up
```

Or add a `Dockerfile`:
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Frontend → Vercel
```bash
# From web/ directory
vercel
```

Set `NEXT_PUBLIC_API_URL` in Vercel environment variables to point to your Railway backend URL.

---

## Quick Start

```bash
# Backend
cd api
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
uvicorn main:app --reload

# Frontend (separate terminal)
cd web
npm install
npm run dev
```

API at `http://localhost:8000`, frontend at `http://localhost:3000`.
