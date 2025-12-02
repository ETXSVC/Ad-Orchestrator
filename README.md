``` markdown
# Creative Ad Generation Orchestrator

A robust, scalable, three-tier microservice designed to modernize and automate the creative ad generation pipeline with mandatory human-in-the-loop approval.

---

## 🌟 Project Overview

The **Creative Ad Generation Orchestrator** replaces a legacy monolithic script with a modern, cloud-native microservice architecture to meet enterprise standards for security, reliability, and scalability. It manages a complete, stateful lifecycle from initial creative brief to final, immutable asset commitment.

## 🚀 Key Features

*   **Three-Tier Microservice Architecture:** Clean separation of concerns across a **Frontend** (user input/approval), a **FastAPI Backend** (orchestration), and **Google Cloud Services** (Persistence, Storage, AI).
*   **Stateful Workflow:** Enforces a mandatory **"Generate » Approve » Commit"** lifecycle, requiring human validation before any asset is permanently stored.
*   **Dual-Modality Generative AI:** Uses a sequential Gemini API process:
    1.  **Image Generation** (Text-to-Image) based on the user's `visual_description_prompt`.
    2.  **Structured Text Generation** (Title, Description, and **exactly 15 SEO Keywords**) based on the newly generated image.
*   **Data Governance & Integrity:**
    *   Structured text output is strictly enforced using **Pydantic Data Models** to guarantee consistency.
    *   The `/commit` endpoint executes an **atomic transaction** to ensure asset upload to GCS and database update are synchronized.
    *   **Cloud SQL (PostgreSQL)** is the mandatory persistence layer for structured metadata, enforcing a rigid schema with 15 flattened keyword columns for analytics.
*   **Layered Security:** Public endpoints (`/generate`, `/commit`) are secured with **API Key Authentication**. Internal communication (to Gemini, GCS, Cloud SQL) uses **Application Default Credentials (ADC)**.

## 🛠️ Technology Stack

| Component | Technology | Purpose |
| :--- | :--- | :--- |
| Backend Framework | **FastAPI** | High-performance, asynchronous orchestration layer. |
| Persistence | **Cloud SQL (PostgreSQL)** | Strong schema enforcement and transactional integrity for metadata. |
| Asset Storage | **Google Cloud Storage (GCS)** | Immutable, UUID-based storage for final creative image assets. |
| Generative Core | **Gemini API** (`gemini-2.5-flash`) | Executes both text-to-image and structured text generation. |
| Data Validation | **Pydantic** | Ensures strict conformity of input requests and AI-generated outputs. |

## ⚙️ Detailed Workflow: Generate » Approve » Commit

1.  **Initiate Generation (`POST /api/v1/generate`):**
    *   Frontend sends creative brief.
    *   Backend generates a unique `ad_id` (UUID).
    *   A 'PENDING' record is written to Cloud SQL.
    *   Sequential Gemini calls (Image then Text) are executed.
    *   Temporary asset links and the `ad_id` are returned for review.
2.  **Human Review:**
    *   Frontend displays the pending image and text content to the user.
    *   User explicitly approves the creative.
3.  **Commitment (`POST /api/v1/commit/{ad_id}`):**
    *   Backend receives the approval signal.
    *   Triggers the **atomic transaction**: permanent GCS upload with unique `ad_id` naming, and final update to the Cloud SQL record (setting `approval_status` to `'APPROVED'`).

---

```
