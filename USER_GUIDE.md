# 📖 USER GUIDE: Automated Cold Email Masterclass

Welcome! Whether you are a first-time user or an automation expert, this guide will walk you through exactly how to keep this system running like a well-oiled machine.

---

## 🏗️ 1. The "Day 0" Setup (One-Time)

Before you send your first email, you need to make sure the environment is ready.

### Step 1: Install the Giants
1. **Docker Desktop**: [Download here](https://www.docker.com/products/docker-desktop/). This is the "Engine" that runs your 6 microservices.
2. **Ollama**: [Download here](https://ollama.com/). This is the "Brain" that runs your local AI.

### Step 2: The .env Secret
Rename `.env.example` to `.env` and fill in your details. 
> [!CAUTION]
> **Gmail App Passwords**: When you generate a password from Google, it will look like `xxxx xxxx xxxx xxxx`. **Remove all spaces** before pasting it into your `.env` file (e.g., `xxxxxxxxxxxxxxxx`). Standard Python tools will crash if those spaces are present!

### Step 3: Add your Resume & Context
1. **The PDF**: Replace `assets/resume.pdf` with your actual resume.
2. **The Bio**: Open `config.yaml` and update your skills, projects, and bio. This is what the AI uses to draft your emails!

### Step 4: Boot the System
```bash
docker compose up -d --build
docker exec -it ollama_llm ollama pull llama3.1:8b
```

---

## 🏃 2. The Daily Routine (Fluent Workflow)

This is how you use the system every day to get new job leads.

### Phase 1: Finding Leads (Apollo.io)
1. Go to [Apollo.io Search](https://app.apollo.io/#/people).
2. Filter for: 
   - **Job Titles**: "Engineering Manager", "HR", "Recruiter", etc.
   - **Industry**: (e.g., "AI", "SaaS", "Fintech").
   - **Email Status**: "Verified".
3. Export the list to a **CSV**.

### Phase 2: The "Hot Folder" Ingest
Instead of using complex commands, just **Drag and Drop** your new Apollo CSV into the **`ingest/`** folder in your project directory. 
- Wait 60 seconds.
- The system will automatically detect the file, import the leads, and rename the file to `.processed`.
- **Done!** Your leads are now in the queue.

### Phase 3: The 10:30 PM Run
- Make sure your laptop is open and Docker is running.
- At **10:30 PM IST**, the n8n scheduler will wake up.
- It will pull 50 leads from the database, have the AI write personalized emails, and send them via Gmail.

---

## 📊 3. Monitoring & Maintenance

### Check your Stats
Want to see how many you've sent today? Open your browser to:
[http://localhost:8002/stats](http://localhost:8002/stats)

### Verify what the AI is writing
If you want to see a "Live Preview" of what the AI is thinking, run this:
```bash
docker logs email_gen --tail 20
```

### Swapping AI Models
- **Local (Default)**: Uses Llama 3.1 8B (Free).
- **Cloud**: Add an `OPENAI_API_KEY` to your `.env` file. The system will automatically detect the key and switch to **GPT-4o-mini** for even faster generation.

---

## 🛠️ 4. Troubleshooting 101

- **"Emails aren't sending!"**: Check if `DRY_RUN=true` is in your `.env`. If it's true, it's just practicing! Set it to `false` for real outreach.
- **"The service is slow!"**: Large AI models take 1-2 minutes per email. This is normal behavior for local machines.
- **"I need to stop the automation!"**:
  ```bash
  docker compose down
  ```

---

> [!TIP]
> **Pro-Tip**: Every week, update your `config.yaml` with a new "Interests" section or a new project you just finished. This keeps your AI-generated emails fresh and relevant to the current market!
