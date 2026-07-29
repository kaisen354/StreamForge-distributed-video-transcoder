<div align="center">

# 🎬 MediaFlow

### _Video Transcoding & AI-Powered Media Editing Platform_

[![Next.js](https://img.shields.io/badge/Next.js-15-black?style=for-the-badge&logo=next.js)](https://nextjs.org/)
[![TypeScript](https://img.shields.io/badge/TypeScript-5-blue?style=for-the-badge&logo=typescript)](https://www.typescriptlang.org/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-336791?style=for-the-badge&logo=postgresql)](https://www.postgresql.org/)
[![Redis](https://img.shields.io/badge/Redis-7-DC382D?style=for-the-badge&logo=redis)](https://redis.io/)
[![AWS](https://img.shields.io/badge/AWS_ECS%2FECR-S3-FF9900?style=for-the-badge&logo=amazon-aws)](https://aws.amazon.com/)
[![Docker](https://img.shields.io/badge/Docker-Containerized-2496ED?style=for-the-badge&logo=docker)](https://www.docker.com/)
[![Cloudinary](https://img.shields.io/badge/Cloudinary-AI_Media-3448C5?style=for-the-badge&logo=cloudinary)](https://cloudinary.com/)
[![Stripe](https://img.shields.io/badge/Stripe-Payments-635BFF?style=for-the-badge&logo=stripe)](https://stripe.com/)

> **MediaFlow** is a production-grade platform that lets you upload raw videos, automatically transcode them into adaptive HLS streams at multiple resolutions, and edit media with a suite of AI-powered tools — all backed by a cloud-native, event-driven architecture on AWS.

</div>

---

## 📌 Table of Contents

- [✨ Features](#-features)
- [🏗️ System Architecture](#️-system-architecture)
- [🔄 User Flow](#-user-flow)
- [🎥 Video Transcoding Pipeline](#-video-transcoding-pipeline)
- [🤖 AI Editing Actions](#-ai-editing-actions)
- [💳 Billing & Subscriptions](#-billing--subscriptions)
- [🛠️ Tech Stack](#️-tech-stack)
- [⚙️ Getting Started](#️-getting-started)
- [📸 Screenshots](#-screenshots)

---

## ✨ Features

| Feature | Description |
|---|---|
| 🔄 **HLS Transcoding** | FFmpeg-powered multi-resolution transcoding to 360p / 480p / 720p adaptive bitrate streams |
| ☁️ **Direct S3 Upload** | Presigned URL flow — browser uploads straight to S3, zero server bandwidth overhead |
| ⚡ **Async Job Queue** | BullMQ + Redis queue decouples upload from processing so the UI never blocks |
| 📡 **Real-time Status** | Redis-backed polling API delivers live QUEUE → PROCESSING → PROCESSED feedback every 5s |
| 🎨 **AI Background Removal** | Cloudinary `e_background_removal` strips backgrounds from images in one click |
| 🌄 **AI Background Replace** | Prompt-driven generative background replacement via Cloudinary AI |
| ✂️ **Smart Video Crop** | Auto-crops video to a target aspect ratio using content-aware gravity fill |
| 🧲 **Object Extraction** | Extract specific objects from images using text prompts with mask/invert support |
| 💬 **Auto Captions** | Google Speech-to-Text via Cloudinary `raw_convert` with burned-in subtitle overlays |
| 💳 **Stripe Subscriptions** | Plan-gated features with Stripe Checkout sessions & Clerk webhook user sync |
| 🐳 **Dockerized Worker** | Isolated ECS task on AWS ECR/ECS — Node 18 + FFmpeg running on Ubuntu Focal |

---

## 🏗️ System Architecture

```mermaid
graph TD
    subgraph Client["🌐 Client — Next.js"]
        UI[Browser UI]
        Upload[Presigned URL Upload]
        Poll["Status Polling\n/api/video/redis-video"]
        Editor[AI Editor Page]
    end

    subgraph NextAPI["⚙️ Next.js API Layer"]
        UploadAPI["/api/video/upload\nGenerates Presigned URL"]
        RedisAPI["/api/video/redis-video\nScans Redis Keys"]
        PaymentAPI["/api/payment\nStripe Checkout"]
        ClerkWebhook["/api/clerk-webhook\nSync User to Postgres"]
    end

    subgraph AWS["☁️ AWS Infrastructure"]
        S3_Input["S3 Input Bucket\nRaw Video Files"]
        S3_Output["S3 Output Bucket\nHLS Streams"]
        ECR["AWS ECR\nDocker Registry"]
        ECS["AWS ECS Task\nProcessing Worker"]
    end

    subgraph Worker["🐳 Docker Worker — Node + FFmpeg"]
        DL["1. Download from S3"]
        FFmpeg["2. FFmpeg Transcode\n360p / 480p / 720p HLS"]
        UL["3. Upload Segments to S3"]
        RedisWrite["4. Write Status to Redis\nvideo:id:status TTL 600s"]
        DBWrite["5. Update Postgres\nstatus = PROCESSED"]
    end

    subgraph DataLayer["🗄️ Data Layer"]
        Redis[("Redis\nJob Status Cache")]
        Postgres[("PostgreSQL\nvia Prisma ORM")]
    end

    subgraph AIEngine["🤖 Cloudinary AI Engine"]
        BGRemove["Background Removal\ne_background_removal"]
        BGReplace["Background Replace\ne_gen_background_replace"]
        SmartCrop["Smart Crop\nc_fill g_auto"]
        Extract["Object Extraction\ne_extract"]
        Caption["Auto Caption\nGoogle Speech raw_convert"]
    end

    UI --> Upload --> UploadAPI --> S3_Input
    S3_Input --> ECS
    ECS --> ECR
    ECS --> Worker
    DL --> FFmpeg --> UL --> S3_Output
    UL --> RedisWrite --> Redis
    RedisWrite --> DBWrite --> Postgres
    Poll --> RedisAPI --> Redis
    Editor --> AIEngine
    UI --> PaymentAPI --> Stripe[("Stripe")]
    ClerkWebhook --> Postgres
```

---

## 🔄 User Flow

```mermaid
flowchart TD
    A(["👤 User Visits MediaFlow"]) --> B{Authenticated?}
    B -- No --> C["Sign In via Clerk"]
    C --> D["Clerk Webhook syncs\nuser to PostgreSQL"]
    D --> E(["Dashboard"])
    B -- Yes --> E

    E --> F{Choose Action}
    F --> G["📤 Upload Video"]
    F --> H["🎨 Open AI Editor"]
    F --> I["💳 Manage Billing"]

    G --> G1["POST /api/video/upload\nGet Presigned URL from S3"]
    G1 --> G2["PUT video binary\ndirectly from Browser to S3"]
    G2 --> G3["createVideo Server Action\nInsert to Postgres — status: QUEUE"]
    G3 --> G4["ECS Task Starts\nDownloads raw video from S3"]
    G4 --> G5["FFmpeg Transcodes\n360p / 480p / 720p HLS"]
    G5 --> G6["HLS Segments uploaded\nto S3 Output Bucket"]
    G6 --> G7["Redis SET\nvideo:id:status = PROCESSED TTL 600s"]
    G7 --> G8["Postgres UPDATE\nstatus = PROCESSED"]
    G8 --> G9{{"✅ Video Ready\nUI polls every 5 seconds"}}

    H --> H1{Select AI Tool}
    H1 --> H2["🧹 Background Removal"]
    H1 --> H3["🌄 Background Replace\nwith text prompt"]
    H1 --> H4["✂️ Smart Crop\nby aspect ratio"]
    H1 --> H5["🧲 Object Extract\nwith prompt + mask"]
    H1 --> H6["💬 Auto Captions\nGoogle Speech-to-Text"]

    H2 & H3 & H4 & H5 & H6 --> H7["Cloudinary builds\ntransformation URL"]
    H7 --> H8["Poll URL every 1s\nuntil Cloudinary responds"]
    H8 --> H9{{"✅ Edited Media Ready"}}

    I --> I1["GET /api/payment\nFetch Stripe plan prices"]
    I1 --> I2["Select plan\nPOST /api/payment"]
    I2 --> I3["Stripe Checkout Session\ncreated server-side"]
    I3 --> I4{{"✅ Subscription Active"}}
```

---

## 🎥 Video Transcoding Pipeline

The transcoding worker is a fully isolated Docker container running on **AWS ECS**. It is triggered per-video, downloads the raw file from S3, runs FFmpeg for each resolution, uploads all HLS segments back to S3, then writes the final status to both Redis and PostgreSQL.

```mermaid
sequenceDiagram
    participant Browser
    participant NextAPI as Next.js API
    participant S3in as S3 Input Bucket
    participant ECS as ECS Worker (Docker + FFmpeg)
    participant S3out as S3 Output Bucket
    participant Redis
    participant Postgres

    Browser->>NextAPI: POST /api/video/upload {filename}
    NextAPI-->>Browser: Presigned PUT URL (expires 900s)
    Browser->>S3in: PUT raw video binary directly

    Browser->>NextAPI: createVideo() Server Action
    NextAPI->>Postgres: INSERT video {status: QUEUE}

    Note over ECS: ECS Task starts with VIDEO_KEY, VIDEO_ID env vars
    ECS->>S3in: GetObject — stream video to disk

    loop Each resolution: 360p, 480p, 720p
        ECS->>ECS: ffmpeg -vf scale=-2:RES -codec:v libx264 -hls_time 10
        ECS->>S3out: PutObject hls_360p.m3u8
        ECS->>S3out: PutObject segment000.ts, segment001.ts ...
    end

    ECS->>Redis: SET video:id:status {PROCESSED, userId} EX 600
    ECS->>Postgres: UPDATE video SET status = PROCESSED

    loop Client polls every 5 seconds
        Browser->>NextAPI: GET /api/video/redis-video
        NextAPI->>Redis: SCAN video:*:status
        Redis-->>Browser: {status: PROCESSED}
    end
```

**HLS Output Structure in S3:**

```
{VIDEO_ID}/
├── output_360p/
│   ├── hls_360p.m3u8        ← Adaptive playlist
│   ├── segment000.ts
│   └── segment001.ts ...
├── output_480p/
│   ├── hls_480p.m3u8
│   └── ...
└── output_720p/
    ├── hls_720p.m3u8
    └── ...
```

---

## 🤖 AI Editing Actions

All AI features are **Next.js Server Actions** powered by **Cloudinary's transformation URL engine**. Each action constructs a transformation URL and polls it (up to 20 attempts, 1s delay) until Cloudinary serves the processed result — no webhooks needed.

| Action | Cloudinary Transformation | Parameters |
|---|---|---|
| **Background Removal** | `e_background_removal` | Image URL |
| **Background Replace** | `e_gen_background_replace:prompt_...` | Image URL + text prompt |
| **Smart Crop** | `ar_{ratio},c_fill,g_auto,h_{height}` | Video URL + aspect ratio + height |
| **Object Extraction** | `e_extract:prompt_(obj);multiple_true;mode_mask` | Image URL + prompt array + mode |
| **Auto Caption** | `raw_convert: google_speech` → subtitle overlay | Cloudinary Video Public ID |

### 💬 Caption Generation Flow

```mermaid
sequenceDiagram
    participant UI
    participant ServerAction as Server Action
    participant Cloudinary
    participant Google as Google Speech-to-Text

    UI->>ServerAction: initiateTranscription(publicId)
    ServerAction->>Cloudinary: api.update(publicId, {raw_convert: "google_speech"})
    Cloudinary->>Google: Audio extraction + Speech-to-Text

    loop Poll every 2s — max 20 attempts
        ServerAction->>Cloudinary: api.resource(publicId)
        Cloudinary-->>ServerAction: info.raw_convert.google_speech.status

        alt status === "complete"
            ServerAction-->>UI: subtitledVideoUrl with overlay transform
        else status === "failed"
            ServerAction-->>UI: error — Transcription failed
        else status === "pending"
            Note over ServerAction: Wait 2s and retry
        end
    end
```

---

## 💳 Billing & Subscriptions

MediaFlow uses **Stripe** for subscription management and **Clerk** for authentication, with automatic user provisioning via webhook.

```mermaid
flowchart LR
    A["👤 User"] --> B["GET /api/payment\nList Stripe Prices"]
    B --> C["Choose Plan"]
    C --> D["POST /api/payment\nCreate Checkout Session\nmode: subscription"]
    D --> E["Stripe Checkout Page"]
    E --> F{Payment Result}
    F -- success --> G["/billing?session_id=...\nConfirmation + Active Plan"]
    F -- cancel --> H["/billing — Try Again"]

    I["Clerk Auth Event"] --> J["/api/clerk-webhook"]
    J --> K["Upsert User in Postgres\nvia Prisma"]
```

---

## 🛠️ Tech Stack

### Frontend
| Technology | Purpose |
|---|---|
| **Next.js 15** | React framework with App Router & Server Actions |
| **TypeScript** | End-to-end type safety across web and worker |
| **Tailwind CSS** | Utility-first styling |
| **ShadCN UI** | Accessible, composable component library |
| **Zustand** | Lightweight global state management |
| **Zod** | Runtime schema validation for Server Actions |

### Backend & Infrastructure
| Technology | Purpose |
|---|---|
| **PostgreSQL + Prisma** | Relational database with type-safe ORM |
| **Redis** | Job status caching with TTL-based expiry |
| **BullMQ** | Reliable background job queue for transcoding |
| **AWS S3** | Raw video input storage + HLS output delivery |
| **AWS ECS + ECR** | Containerised FFmpeg worker deployment |
| **FFmpeg** | Video transcoding inside Docker on Ubuntu Focal |
| **Cloudinary** | AI image/video transformation engine |
| **Clerk** | Authentication + webhook-driven user management |
| **Stripe** | Subscription billing and checkout sessions |

---

## ⚙️ Getting Started

### Prerequisites

- Node.js 18+
- Docker
- AWS CLI configured with ECS/ECR/S3 access
- PostgreSQL database
- Redis instance
- Cloudinary, Clerk, and Stripe accounts

### 1. Clone the Repository

```bash
git clone https://github.com/your-username/media-flow.git
cd media-flow
```

### 2. Setup the Web App

```bash
cd web
cp .env.sample .env
# Fill in all environment variables (see below)
npm install
npx prisma migrate dev
npm run dev
```

### 3. Setup the Video Processing Worker

```bash
cd video-processing
cp .env.example .env
# Fill in AWS credentials and video config
npm install
```

### 4. Build & Deploy Docker Worker to AWS ECR/ECS

```bash
# Build the image (includes FFmpeg)
docker build -t mediaflow-worker ./video-processing

# Tag and push to ECR
docker tag mediaflow-worker:latest <your-ecr-uri>:latest
aws ecr get-login-password | docker login --username AWS --password-stdin <your-ecr-uri>
docker push <your-ecr-uri>:latest

# Register ECS Task Definition and run as needed
```

### Environment Variables

**`web/.env`**
```env
DATABASE_URL=postgresql://user:pass@host/db
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_...
CLERK_SECRET_KEY=sk_...
CLOUDINARY_KEY=your_cloudinary_api_key
CLOUDINARY_SECRET=your_cloudinary_api_secret
VIDEO_BUCKET=your-s3-input-bucket
VIDEO_UPLOAD_KEY=uploads
TARGET_BUCKET_NAME=your-s3-output-bucket
STRIPE_SECRET=sk_live_...
NEXT_PUBLIC_URL=http://localhost:3000
REDIS_URL=redis://localhost:6379
```

**`video-processing/.env`**
```env
AWS_REGION=us-east-1
AWS_ACCESS_KEY=AKIA...
AWS_SECRET_ACCESS_KEY=...
VIDEO_BUCKET=your-s3-input-bucket
VIDEO_KEY=uploads/your-video.mp4
VIDEO_ID=<postgres-video-uuid>
TARGET_BUCKET_NAME=your-s3-output-bucket
TARGET_BUCKET_KEY=processed
AWS_CLUSTER_ARN=arn:aws:ecs:us-east-1:123:cluster/...
ECS_TASK_ARN=arn:aws:ecs:us-east-1:123:task/...
DATABASE_URL=postgresql://...
REDIS_URL=redis://...
```

---

## 📸 Screenshots

<div align="center">

![Landing Page](web/public/screenshots/1_IMG_1.png)

![Dashboard](web/public/screenshots/2_IMG_2.png)

![Video Upload Workflow](web/public/screenshots/2_IMG_2_2.png)

![AI Editor](web/public/screenshots/3_IMG_3.png)

![Billing](web/public/screenshots/4_IMG_4.png)

![Processed Videos](web/public/screenshots/5_IMG_5.png)

![Settings](web/public/screenshots/6_IMG_6.png)

</div>

---

## 🤝 Contributing

Contributions are welcome! Feel free to open a pull request or file an issue for bugs, feature suggestions, or improvements.

---

## 🙏 Acknowledgments

- **Cloudinary** — exceptional AI image/video processing APIs
- **AWS** — scalable cloud storage and container orchestration
- **Prisma + PostgreSQL** — seamless and type-safe database management
- **Stripe** — developer-friendly subscription billing

---

<div align="center">

Built with ❤️ using Next.js · FFmpeg · AWS · Cloudinary AI

⭐ **Star this repo if you found it useful!**

</div>
