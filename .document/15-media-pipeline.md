# 15 — Media Pipeline

**Source files**: `src/media/`, `src/media-understanding/`

---

## 1. Overview

OpenClaw's media pipeline has two layers:

| Layer | Module | Responsibility |
|-------|--------|---------------|
| **Media I/O** | `src/media/` | Parse, validate, transcode, host, serve media files |
| **Media Understanding** | `src/media-understanding/` | AI-powered transcription, description, and video analysis |

When a user sends an image, audio file, video, or document, it flows through both layers before the agent sees it.

---

## 2. Media I/O Layer (`src/media/`)

### 2.1 Parsing

`src/media/parse.ts` — extracts `MEDIA:` tokens from message text:
- Recognizes local file paths, remote URLs, and base64-encoded data
- Validates extensions against MIME allowlists
- Enforces a 4,096-character limit on media source strings

### 2.2 Input File Validation

`src/media/input-files.ts` — validates inbound media files:

```typescript
type InputFileLimits = {
  allowedMimes: string[];   // e.g. ["image/png", "image/jpeg", ...]
  maxBytes: number;         // file size limit
  maxChars: number;         // character limit for text extraction
};
```

File types handled:
- **Images**: JPEG, PNG, WebP, GIF (validated + optionally resized)
- **Audio**: MP3, WAV, OGG, M4A (transcoded via FFmpeg if needed)
- **Video**: MP4, MOV, WebM
- **Documents**: PDF (text/image extracted), plain text

### 2.3 Image Operations

`src/media/image-ops.ts` — image processing via [Sharp](https://sharp.pixelplumbing.com/):
- Resize to platform-safe dimensions (e.g., Telegram has 5MB image limit)
- Convert between formats (WebP → JPEG for platforms that don't support WebP)
- Strip EXIF metadata for privacy

### 2.4 Audio Processing

`src/media/audio.ts` — audio via FFmpeg:
- Transcode to WAV/MP3 for transcription providers
- Read and strip audio tags (artist, title, etc.)
- Normalize sample rate and channels

### 2.5 FFmpeg Integration

`src/media/ffmpeg-exec.ts` — safe FFmpeg subprocess execution:
- Enforces duration and output size limits
- Streams output via async iterables
- Respects exec approval policy (FFmpeg is on the safe-bin list)

### 2.6 PDF Extraction

`src/media/pdf-extract.ts` — extracts content from PDFs:
- Text content via `pdfjs-dist`
- Embedded images for visual understanding

### 2.7 Media Hosting

`src/media/host.ts` — ensures media is accessible to external AI providers that need a URL:
- Spins up a temporary media server on port 42873
- Generates short-TTL media URLs (the file is served for only the duration needed)
- Tunnels via Tailscale or existing webhook server URL if available

```
Media file on disk
    │
    ▼
media server (port 42873)
    │  temporary TTL URL
    ▼
AI provider (Gemini, etc.) fetches URL
```

---

## 3. Media Understanding Layer (`src/media-understanding/`)

### 3.1 Overview

`src/media-understanding/apply.ts` — `applyMediaUnderstanding()` is the main orchestrator. For each message with attachments:
1. Resolves which capabilities apply (image, audio, video)
2. Selects providers for each capability
3. Runs providers concurrently
4. Formats results as structured text for the agent

### 3.2 Understanding Flow

```
Message with attachments
    │
    ▼
runner.entries.ts — entry resolution
    │  Normalize attachments, validate scope, apply limits
    ▼
apply.ts — capability dispatch
    ├── images → image provider(s)
    ├── audio  → audio transcription provider(s)
    └── video  → video understanding provider(s)
    │
    ▼ (concurrent execution)
Provider results collected
    │
    ▼
format.ts — format as XML/text for agent context
    │
    ▼
Injected into agent's message context
```

### 3.3 Supported Providers

**Image understanding**:
| Provider | Notes |
|----------|-------|
| Claude (Anthropic) | via vision API |
| Gemini | Google Generative AI |
| Azure OpenAI | via GPT-4V |
| Ollama | local self-hosted vision models |

**Audio transcription**:
| Provider | Notes |
|----------|-------|
| Google (Gemini) | `src/media-understanding/providers/google/audio.ts` |
| Deepgram | `src/media-understanding/providers/deepgram/audio.ts` |
| Whisper (via Ollama) | local transcription |

**Video understanding**:
| Provider | Notes |
|----------|-------|
| Google (Gemini) | `src/media-understanding/providers/google/video.ts` |
| Moonshot | `src/media-understanding/providers/moonshot/video.ts` |

### 3.4 Key Types

```typescript
type MediaUnderstandingCapability = "image" | "audio" | "video";

type MediaUnderstandingOutput = {
  kind: MediaUnderstandingCapability;
  attachmentIndex: number;
  text: string;       // transcription or description
  provider: string;
  model: string;
};

type ApplyMediaUnderstandingResult = {
  outputs: MediaUnderstandingOutput[];
  decisions: MediaUnderstandingDecision[];
  appliedImage: boolean;
  appliedAudio: boolean;
  appliedVideo: boolean;
  appliedFile: boolean;
};
```

### 3.5 Concurrency and Caching

`src/media-understanding/resolve.ts` — concurrency control:
- Semaphore limits parallel provider calls (prevents API rate limits)
- Configurable timeouts per provider
- Cached results: same attachment in the same session is not re-processed

### 3.6 Scope Control

`src/media-understanding/scope.ts` — controls which sessions/channels can use media understanding:
- Can be restricted to specific channels or agent IDs
- Config key: `media.understanding.*`

### 3.7 Output Format

`src/media-understanding/format.ts` — formats understanding outputs as XML-tagged sections injected into the agent's system context:

```xml
<media_understanding>
  <image index="0" provider="gemini" model="gemini-2.0-flash">
    A screenshot showing a Python error traceback in a terminal...
  </image>
  <audio index="1" provider="deepgram" model="nova-3">
    "Please help me fix the bug in my authentication module."
  </audio>
</media_understanding>
```

The agent sees this as part of the message context, treating it as first-class information.

---

## 4. SSRF Protection

Both layers apply SSRF (Server-Side Request Forgery) protection:
- `src/plugin-sdk/ssrf-policy.ts` validates all outbound media URLs before fetching
- Blocks internal IP ranges (127.x, 10.x, 172.16.x, 192.168.x, link-local, etc.)
- Applied before any external URL is fetched for transcoding or understanding

---

## 5. Key Files Summary

| File | Role |
|------|------|
| `src/media/parse.ts` | MEDIA token extraction and validation |
| `src/media/input-files.ts` | File type validation, MIME allowlists, size limits |
| `src/media/image-ops.ts` | Image resize and format conversion (Sharp) |
| `src/media/audio.ts` | Audio processing (FFmpeg) |
| `src/media/ffmpeg-exec.ts` | Safe FFmpeg subprocess execution |
| `src/media/pdf-extract.ts` | PDF text and image extraction |
| `src/media/host.ts` | Temporary media URL server |
| `src/media/store.ts` | Media file storage management |
| `src/media/constants.ts` | MIME types and defaults |
| `src/media-understanding/apply.ts` | Main understanding orchestrator |
| `src/media-understanding/runner.entries.ts` | Attachment entry resolution |
| `src/media-understanding/types.ts` | Provider and output type definitions |
| `src/media-understanding/audio-transcription-runner.ts` | Audio pipeline with fallback |
| `src/media-understanding/format.ts` | XML output formatting |
| `src/media-understanding/resolve.ts` | Concurrency and timeout resolution |
| `src/media-understanding/scope.ts` | Access scope control |
| `src/media-understanding/providers/` | Provider implementations |
