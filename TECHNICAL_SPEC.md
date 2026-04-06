# TECHNICAL SPECIFICATION: Automated Cold Email Engine

This document provides a deep-dive into the architecture, design decisions, and implementation details of the Automated Cold Email system. It is intended for developers and engineers who want to understand, maintain, or scale this platform.

---

## 1. System Design Philosophy

The core of this project is built on **Decoupled Microservices**. Instead of a single, monolithic script, we use separate specialized containers.

### Why Microservices?
- **Isolation**: If the Email Sender crashes (e.g., due to an SMTP error), it doesn't stop the Contacts Service from ingesting new leads.
- **Resource Management**: The `ollama_llm` container is extremely heavy (8GB+ RAM). By keeping it separate, we can restart or upgrade it without touching the business logic.
- **Scalability**: We could theoretically run 10 instances of `email_gen` across different machines if we needed to process 10,000 emails per hour.

### Why n8n?
n8n acts as the **System Brain (Orchestrator)**. It is essentially a "Visual State Machine." It handles:
- **Scheduling**: The "Cron" trigger that wakes up the system.
- **Flow Control**: The `ForEach` loop that ensures we process contacts one-by-one.
- **Error Handling**: Branching logic that marks a contact as `failed` if an HTTP call returns anything other than a `200 OK`.

---

## 2. Microservice Deep-Dive

### 2.1 Contacts Service (`contacts_svc` - Port 8000)
**Role**: Data Persistence and Queue Management.

- **Storage**: Uses **SQLite3**. Unlike a complex database like PostgreSQL, SQLite is a single file (`contacts.db`). This makes it perfect for local automation.
- **CSV Ingestion Algorithm**: 
    1. The `/upload-csv` endpoint receives an Apollo export.
    2. It uses Python's native `csv.DictReader` to map column headers (`First Name`, `Email`, etc.) to our internal Schema.
    3. It performs an `INSERT OR IGNORE` operation. This is our **de-duplication engine**—the `email` column is marked as `UNIQUE`, so the same lead is never added twice.
- **Queue Logic**: n8n hits `/fetch-contacts` with a `limit` (e.g., 50). The service runs `SELECT * FROM contacts WHERE status = 'pending'`. This ensures we always pick up where we left off.

### 2.2 Email Generator (`email_gen` - Port 8001)
**Role**: Personalization Logic (LLM Interface).

- **Framework**: **LangChain**. We use LangChain to manage the interaction between our Python code and the Ollama model.
- **Prompt Engineering**: 
    - The **System Prompt** defines the persona ("Professional Job Seeker"). 
    - The **Context Injection** reads your `config.yaml` at startup. This context is sent with every single request to the LLM, ensuring the AI "knows" your career history.
- **JSON Enforcement**: We use a `JsonOutputParser`. This is critical. The LLM must return a valid JSON object with `subject` and `body` keys so that n8n can programmatically read them.

### 2.3 Email Sender (`email_send` - Port 8002)
**Role**: Delivery and Compliance.

- **SMTP Protocol**: Uses `smtplib` and `email.mime`. 
    - **Header UTF-8 Encoding**: We use `email.header.Header` to ensure non-ASCII characters in subjects don't crash the server.
- **Attachment Logic**: It dynamically reads `/app/assets/resume.pdf` from the container's volume and encodes it as Base64 (MIME application/pdf) for the email.
- **Rate Limiting**: 
    - Stores state in `rate_limiter.json`. 
    - Before every send, it checks if `today_count < DAILY_LIMIT`. 
    - This is the most important "Security" feature to prevent your Gmail account from being flagged for spam.

---

## 3. Network Topology

All services are connected via a Docker Virtual Network called `automation_net`. 

- **Internal DNS**: Inside the network, services talk to each other using their service names (`http://email_gen:8001`) rather than local IP addresses.
- **External Access**: Only `n8n` (5678) and the service ports (8000, 8001, 8002) are mapped to your Mac's `localhost`. The `ollama` and `n8n_db` services remain hidden behind the network for security.

---

## 4. Privacy & Data Flow

1. **Air-Gapped AI**: Your career data stays in the `ollama_llm` Docker volume. It is never uploaded to the cloud (no OpenAI/Anthropic involvement).
2. **Environment Protection**: Sensitive keys are stored in `.env`. The `.gitignore` prevents these from ever reaching GitHub.
3. **Local State**: The `contacts.db` is your "Gold Record." Even if the internet cuts out, your progress is saved locally.

---

## 5. Scaling for the Future

**How to send 1,000+ emails?**
1. **Multiple Gmails**: You would need to rotate `SMTP_USER` credentials.
2. **Dedicated VPS**: Moving this to a cloud server (like DigitalOcean or AWS) would allow it to run 24/7 without your laptop being open.
3. **CRM Integration**: Instead of SQLite, you could connect the n8n "Log Success" node directly to a Google Sheet or Airtable for visual tracking.
