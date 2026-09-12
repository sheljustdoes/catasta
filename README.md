# catasta

Two ways to turn a research pipeline into a polished, interactive demo without shipping the pipeline itself to the browser.

catasta pairs a Python backend (where the science lives, and stays server-side) with a Next.js + TypeScript frontend (where people experience it). It comes in two variants:

- **Variant A — Embedded.** One portfolio site, one shared backend, demos as routes. Default choice.
- **Variant B — Standalone.** Forked per project, its own frontend + backend deployment, its own URL.

The pipeline pattern (`preprocess → predict → postprocess`, structured request/response schemas, upload-and-reveal UI) is identical in both — only the deployment topology differs.

---

## Choosing a variant

| | Variant A — Embedded | Variant B — Standalone |
|---|---|---|
| Frontend | A route inside the existing portfolio site | Its own Next.js app, its own repo |
| Backend | One shared FastAPI service, one router per project | Its own FastAPI service |
| Hosting | Frontend: GitHub Pages (static export). Backend: one small Railway/Render instance | Frontend: Vercel. Backend: its own Railway/Render instance |
| Cost/upkeep | One backend to pay for and monitor, regardless of demo count | One backend per project |
| Use when | The demo just needs to show input → output behind an API boundary | The project needs independent scaling (e.g. GPU inference), its own domain/identity, or is being spun out separately from the portfolio |

Default to **Variant A**. Reach for **Variant B** only when a specific project actually needs to stand alone — most demos don't.

---

## Variant A — Embedded (default)

### Architecture

```
sheljustdoes.github.io/            # the portfolio site itself
├── app/
│   ├── projects/
│   │   ├── iridis/
│   │   │   └── page.tsx           # demo route — calls /api/iridis/predict
│   │   ├── topos/
│   │   │   └── page.tsx
│   │   └── veridian/
│   │       └── page.tsx
│   ├── layout.tsx
│   └── page.tsx                   # résumé / landing
├── components/
│   ├── upload/Dropzone.tsx        # shared across demo routes
│   └── results/ResultCard.tsx
├── lib/
│   ├── api.ts                     # client for the shared backend
│   └── types.ts
└── next.config.ts                 # output: 'export', images.unoptimized: true

demo-backend/                       # one small service, separate repo
├── main.py                         # mounts one router per project
├── routers/
│   ├── iridis.py
│   ├── topos.py
│   └── veridian.py
├── services/
│   ├── iridis_pipeline.py          # each project's Pipeline class
│   ├── topos_pipeline.py
│   └── veridian_pipeline.py
├── core/
│   ├── config.py
│   └── models.py                   # schemas, namespaced per project
├── requirements.txt
└── Dockerfile
```

Two repos instead of one, but only ever *two* — the frontend repo is the portfolio site you already have, and the backend repo grows by one router + one pipeline module per new demo rather than by one whole deployment.

### Backend — one service, routed per project

`main.py`:

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from routers import iridis, topos, veridian
from core.config import settings

app = FastAPI(title="demo-backend", version=settings.VERSION)

app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.ALLOWED_ORIGINS,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

app.include_router(iridis.router, prefix="/api/iridis", tags=["iridis"])
app.include_router(topos.router, prefix="/api/topos", tags=["topos"])
app.include_router(veridian.router, prefix="/api/veridian", tags=["veridian"])


@app.get("/health")
async def health():
    return {"status": "ok"}
```

`routers/iridis.py` (same shape for every project — this is the file that repeats):

```python
import time
from fastapi import APIRouter, UploadFile, File, HTTPException
from core.models import PredictionResponse
from services.iridis_pipeline import IridisPipeline

router = APIRouter()
pipeline = IridisPipeline(model_path="./models/iridis", device="cpu")


@router.post("/predict", response_model=PredictionResponse)
async def predict(file: UploadFile = File(...)):
    if not file.content_type.startswith("image/"):
        raise HTTPException(status_code=400, detail="Expected an image file")

    start = time.time()
    contents = await file.read()
    results = pipeline.run(contents)

    return PredictionResponse(
        project="iridis",
        results=results,
        processing_time_ms=round((time.time() - start) * 1000, 2),
    )
```

`services/iridis_pipeline.py` follows the same `preprocess → predict → postprocess` shape as Variant B's `Pipeline` class below — see that section for the full pattern; only the class name and module location change (one file per project instead of one file, period).

### Frontend — a route, not a repo

`app/projects/iridis/page.tsx` is Variant B's `app/page.tsx` (below) with two changes: it's nested under `app/projects/iridis/` instead of being the site root, and its fetch call points at `/api/iridis/predict` on the shared backend instead of `/api/v1/predict` on a dedicated one. The `Dropzone` and `ResultCard` components are shared verbatim across every project route — write them once, import everywhere.

`lib/api.ts`:

```typescript
const API_BASE = process.env.NEXT_PUBLIC_API_URL!; // one shared backend URL

export async function predictIridis(file: File) {
  const formData = new FormData();
  formData.append("file", file);
  const res = await fetch(`${API_BASE}/api/iridis/predict`, { method: "POST", body: formData });
  if (!res.ok) throw new Error((await res.json().catch(() => ({}))).detail ?? `API error: ${res.status}`);
  return res.json();
}
// predictTopos, searchVeridian, etc. follow the same shape.
```

### Deployment

**Backend → Railway or Render**, once, same as Variant B:

```bash
railway init
railway up
```

**Frontend → GitHub Pages**, via static export instead of Vercel:

```ts
// next.config.ts
const nextConfig = {
  output: "export",
  images: { unoptimized: true },   // no Image Optimization server on static hosts
};
export default nextConfig;
```

A GitHub Actions workflow (`.github/workflows/deploy-pages.yml`) builds and publishes `out/` on every push to `main` — no manual deploy step, no Vercel account. Set `NEXT_PUBLIC_API_URL` as a repository variable so it's baked in at build time.

### Adding a new demo

1. Add `routers/<project>.py` + `services/<project>_pipeline.py` to the shared backend; mount the router in `main.py`.
2. Add `app/projects/<project>/page.tsx` to the site, reusing `Dropzone`/`ResultCard`.
3. Push both. No new deployment target.

---

## Variant B — Standalone

Use this when a project needs to be its own thing: independent scaling, its own domain, or a deployment lifecycle decoupled from the rest of the portfolio.

Forked per project — swap the pipeline, adjust the UI, deploy. The architecture stays the same across forks; the research changes.

### Architecture

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

### API — FastAPI Backend

#### `api/main.py`

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

#### `api/core/config.py`

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

#### `api/core/models.py`

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

#### `api/routes/predict.py`

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

#### `api/services/pipeline.py`

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

### Frontend — Next.js + TypeScript + Tailwind

#### `web/lib/types.ts`

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

#### `web/lib/api.ts`

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

#### `web/app/layout.tsx`

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
  title: "Project Demo",
  description: "One-line description of what this demo shows.",
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

#### `web/app/page.tsx`

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
        <h1 className="text-3xl font-semibold tracking-tight">Project Name</h1>
        <p className="mt-2 text-neutral-500">One-line description</p>
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

#### `web/components/upload/Dropzone.tsx`

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

#### `web/components/results/ResultCard.tsx`

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

### Adapting per project

#### Backend — what to change:

| File | What to swap |
|------|-------------|
| `core/config.py` | `PROJECT_NAME`, `PROJECT_DESCRIPTION`, `MODEL_PATH` |
| `core/models.py` | Request/response schemas to match your pipeline's I/O |
| `services/pipeline.py` | The actual science — load, preprocess, predict, postprocess |
| `routes/predict.py` | Input type (`UploadFile` for images, `Body` for JSON, etc.) |

#### Frontend — what to change:

| File | What to swap |
|------|-------------|
| `app/layout.tsx` | Title, description, font choice |
| `app/page.tsx` | Header text, input component (Dropzone vs text input vs form) |
| `components/results/ResultCard.tsx` | Result display (cards, charts, visualizations) |
| `lib/types.ts` | Type definitions to match new API response shape |
| `lib/api.ts` | Endpoint path if changed; input encoding if not FormData |

#### Input patterns per project type:

| Project type | Input | Component |
|---------|-------|-----------|
| Image classification/clustering | Image upload | `Dropzone` (image/*) |
| Genomic/tabular data | File upload | `Dropzone` (accept=".csv,.vcf") |
| Literature/text retrieval | Search query | Text input + submit |
| Conversational/agentic | Conversation | Text area + submit |

### Deployment

#### Backend → Railway (or Render)
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

#### Frontend → Vercel
```bash
# From web/ directory
vercel
```

Set `NEXT_PUBLIC_API_URL` in Vercel environment variables to point to your Railway backend URL.

### Quick Start

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

---

## Status

Both variants are specifications, not scaffolded code yet — this README is the source of truth until a project actually needs one instantiated. First real build will be Variant A, once the iridis segmentation rework (in progress) lands.
