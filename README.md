# 🤖 JobHunterAgent

An AI-powered job application automation system that generates tailored resumes, auto-fills application forms on major ATS platforms (Greenhouse & Lever), and integrates with n8n workflows — all with human-in-the-loop approval before submission.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Prerequisites](#prerequisites)
- [Step-by-Step Setup](#step-by-step-setup)
  - [1. Clone the Repository](#1-clone-the-repository)
  - [2. Install Dependencies](#2-install-dependencies)
  - [3. Install Playwright Browsers](#3-install-playwright-browsers)
  - [4. Configure Your Profile](#4-configure-your-profile)
  - [5. Set Up Environment Variables](#5-set-up-environment-variables)
  - [6. Set Up n8n (Workflow Automation)](#6-set-up-n8n-workflow-automation)
  - [7. Start the Server](#7-start-the-server)
- [How It Works](#how-it-works)
- [Chrome Extension (Job Cyborg)](#chrome-extension-job-cyborg)
  - [Installing the Extension](#installing-the-extension)
  - [Using the Extension](#using-the-extension)
- [Supported ATS Platforms](#supported-ats-platforms)
- [API Reference](#api-reference)
- [Project Structure](#project-structure)
- [Troubleshooting](#troubleshooting)
- [License](#license)

---

## Overview

JobHunterAgent is a two-part system:

1. **Local Node.js Server + Playwright Agent** — Receives job data from n8n, generates a tailored PDF resume from AI-written markdown, opens a real browser, navigates to the application page, auto-fills the form, and waits for your approval before submitting.

2. **Job Cyborg Chrome Extension** — A browser extension you can click on any job page. It scrapes the form fields and job description, sends them to n8n for AI processing, and auto-fills the answers directly in your browser.

---

## Architecture

```
┌─────────────────┐     ┌──────────┐     ┌──────────────────────┐
│   n8n Workflow   │────▶│  Server  │────▶│   Playwright Agent   │
│  (AI + Sheets)   │     │ :3000    │     │  (Visible Browser)   │
└─────────────────┘     └──────────┘     └──────────────────────┘
        │                                         │
        │                                         ▼
        │                                ┌──────────────────┐
        │                                │  ATS Form Filler │
        │                                │  Greenhouse/Lever│
        │                                └──────────────────┘
        │                                         │
        ▼                                         ▼
┌─────────────────┐                      ┌──────────────────┐
│  Status Webhook  │◀────────────────────│  Human Approval  │
│  (back to n8n)   │                     │  (Terminal y/m/s)│
└─────────────────┘                      └──────────────────┘

┌─────────────────────────────────────────────────────────┐
│              Job Cyborg Chrome Extension                │
│  popup.html → content.js → background.js → n8n → fill  │
└─────────────────────────────────────────────────────────┘
```

---

## Tech Stack

| Component          | Technology                     |
| ------------------ | ------------------------------ |
| Runtime            | Node.js (CommonJS)             |
| Web Framework      | Express 5                      |
| Browser Automation | Playwright (Chromium)          |
| PDF Generation     | Playwright (headless Chromium) |
| Workflow Engine    | n8n (self-hosted)              |
| Chrome Extension   | Manifest V3                    |
| Environment Config | dotenv                         |

---

## Prerequisites

Before you begin, make sure you have the following installed:

- **Node.js** (v18 or higher) — [Download](https://nodejs.org/)
- **n8n** (self-hosted) — [Install Guide](https://docs.n8n.io/hosting/)
- **Google Chrome** (for the extension)
- **Git** (optional, for cloning)

---

## Step-by-Step Setup

### 1. Clone the Repository

```bash
git clone https://github.com/YOUR_USERNAME/JobHunterAgent.git
cd JobHunterAgent
```

### 2. Install Dependencies

```bash
npm install
```

This installs:
- `express` — HTTP server
- `playwright` — browser automation & PDF generation
- `dotenv` — environment variable management

### 3. Install Playwright Browsers

Playwright needs a Chromium browser binary to work:

```bash
npx playwright install chromium
```

> **Tip:** If you want to store browsers in a custom location, set the `PLAYWRIGHT_BROWSERS_PATH` environment variable (see Step 5).

### 4. Configure Your Profile

Open **`config.js`** and update it with your personal information:

```js
module.exports = {
  profile: {
    firstName: "YourFirstName",
    lastName: "YourLastName",
    fullName: "Your Full Name",
    email: "your.email@example.com",
    phone: "+1234567890",            // International format
    location: "City, Country",
    linkedin: "https://www.linkedin.com/in/your-profile",
    github: "https://github.com/your-username",
    workAuthorization: "Yes",        // or "No"
    requiresSponsorship: "No"        // or "Yes"
  }
};
```

This profile data is used to auto-fill application forms.

### 5. Set Up Environment Variables

Create a **`.env`** file in the project root (or edit the existing one):

```env
PORT=3000
N8N_WEBHOOK_URL=http://localhost:5678/webhook/application-status
PLAYWRIGHT_BROWSERS_PATH=D:\pw-browsers
```

| Variable                   | Description                                                    |
| -------------------------- | -------------------------------------------------------------- |
| `PORT`                     | Port the Express server listens on (default: `3000`)           |
| `N8N_WEBHOOK_URL`          | The n8n webhook URL that receives status updates after applying |
| `PLAYWRIGHT_BROWSERS_PATH` | (Optional) Custom path for Playwright browser binaries         |

### 6. Set Up n8n (Workflow Automation)

1. **Start n8n** (if not already running):
   ```bash
   npx n8n
   ```
   This starts n8n on `http://localhost:5678`.

2. **Create a workflow** in n8n that:
   - Scrapes job listings (e.g., from Google Sheets, Remote OK, etc.)
   - Uses an AI node (e.g., OpenAI) to generate a **tailored resume in Markdown** and a **cover letter** for each job
   - Sends a `POST` request to `http://localhost:3000/apply` with the job data
   - Has a **webhook node** at `/webhook/application-status` to receive results back

3. **Expected payload** your n8n workflow should send to the agent:

   ```json
   {
     "rowId": 51,
     "applyUrl": "https://boards.greenhouse.io/company/jobs/12345",
     "resumeMarkdown": "# Your Name\n## Software Engineer\n...",
     "coverLetter": "Dear Hiring Manager, ..."
   }
   ```

### 7. Start the Server

```bash
node server.js
```

You should see:

```
Local Job Agent is awake and listening on http://localhost:3000
```

The agent is now ready to receive job applications from n8n! 🎉

---

## How It Works

Here's what happens when a job application is triggered:

1. **n8n sends job data** → `POST http://localhost:3000/apply`
2. **Server immediately responds** with `202 Accepted` (so n8n doesn't time out)
3. **PDF Resume is generated** from the AI-written markdown using Playwright (saved to `resumes/resume_{rowId}.pdf`)
4. **Account-gated boards are detected** (e.g., We Work Remotely) — these are skipped with status `"Needs Human Input"`
5. **Playwright opens a visible Chromium browser** and navigates to the application URL
6. **CAPTCHA detection** — if a CAPTCHA is found, the agent halts
7. **ATS platform is identified** (Greenhouse or Lever) and the appropriate form filler runs
8. **Form is auto-filled** with your profile data, resume PDF, and cover letter
9. **You review the form** in the open browser window
10. **Terminal prompts for approval:**
    - `y` → Agent clicks the submit button automatically
    - `m` → You already submitted manually
    - `s` → Skip/cancel this application
11. **Result is reported back** to n8n via the callback webhook

---

## Chrome Extension (Job Cyborg)

The **Job Cyborg** Chrome Extension is a companion tool that lets you auto-fill job application forms from any page with one click.

### Installing the Extension

1. Open Chrome and navigate to `chrome://extensions/`
2. Enable **Developer mode** (toggle in the top-right)
3. Click **"Load unpacked"**
4. Select the `Job-Cyborg-Extension` folder from this project
5. The **Job Cyborg** extension icon will appear in your toolbar

### Using the Extension

1. Navigate to any job application page (Greenhouse, Lever, Workday, etc.)
2. Click the **Job Cyborg** extension icon in your toolbar
3. Click the green **"Auto-Apply"** button
4. The extension will:
   - Scrape all visible form fields and the job description from the page
   - Send them to your n8n workflow for AI processing
   - Receive back AI-generated answers
   - Auto-fill the form fields directly in your browser
5. Review the filled fields and submit manually

> **Note:** The extension communicates with n8n at `http://localhost:5678/webhook-test/cyborg-apply`. Make sure you have a matching webhook node in your n8n workflow.

---

## Supported ATS Platforms

| Platform        | Auto-Fill | Auto-Submit | Notes                                       |
| --------------- | --------- | ----------- | ------------------------------------------- |
| **Greenhouse**  | ✅        | ✅          | Full support: personal info, dropdowns, file upload, custom questions |
| **Lever**       | ✅        | ✅          | Full support: name, email, phone, links, resume, cover letter |
| **Other**       | ❌        | ❌          | Returns `"Needs Human Input"` — resume PDF is still generated |

### Account-Gated Boards

Some job boards require account creation before revealing the real application link. These are automatically detected and skipped:

- **We Work Remotely** — Requires login to access employer links

---

## API Reference

### `POST /apply`

Triggers the job application pipeline.

**Request Body:**

```json
{
  "rowId": 51,
  "applyUrl": "https://boards.greenhouse.io/company/jobs/12345",
  "resumeMarkdown": "# Full Name\n## Role\n- Experience...",
  "coverLetter": "Dear Hiring Manager, I am excited to apply..."
}
```

| Field            | Type   | Required | Description                            |
| ---------------- | ------ | -------- | -------------------------------------- |
| `rowId`          | number | Yes      | Unique row identifier (from your sheet) |
| `applyUrl`       | string | Yes      | Direct URL to the job application form  |
| `resumeMarkdown` | string | Yes      | AI-tailored resume content in Markdown  |
| `coverLetter`    | string | No       | Cover letter text                       |

**Response:** `202 Accepted`

```json
{
  "message": "Application process queued locally",
  "rowId": 51
}
```

**Callback to n8n** (after processing):

```json
{
  "rowId": 51,
  "status": "Applied",
  "resumePdfPath": "D:\\JobHunterAgent\\resumes\\resume_51.pdf",
  "reason": null,
  "error": null
}
```

Possible `status` values:
- `"Applied"` — Successfully submitted
- `"Needs Human Input"` — Form filled but needs manual review/submission
- `"CAPTCHA"` — CAPTCHA detected, requires manual solving
- `"Failed"` — An error occurred during automation

---

## Project Structure

```
JobHunterAgent/
├── server.js                     # Express server — receives jobs from n8n
├── agent.js                      # Core Playwright automation pipeline
├── config.js                     # Your personal profile data
├── pdfGenerator.js               # Converts AI markdown resume → styled PDF
├── .env                          # Environment variables (PORT, webhooks)
├── package.json                  # Node.js dependencies
│
├── ats/                          # ATS-specific form fillers
│   ├── greenhouse.js             # Greenhouse.io form automation
│   └── lever.js                  # Lever.co form automation
│
├── Job-Cyborg-Extension/         # Chrome Extension (Manifest V3)
│   ├── manifest.json             # Extension configuration
│   ├── background.js             # Service worker — routes messages to n8n
│   ├── content.js                # Content script — scrapes & fills forms
│   ├── popup.html                # Extension popup UI
│   └── popup.js                  # Popup button logic
│
└── resumes/                      # Generated PDF resumes (auto-created)
    ├── resume_51.pdf
    ├── resume_52.pdf
    └── ...
```

---

## Troubleshooting

### "Cannot find Chromium" error
Make sure Playwright browsers are installed:
```bash
npx playwright install chromium
```

### n8n webhook not responding
- Ensure n8n is running (`npx n8n` or your Docker setup)
- Check that the webhook URL in `.env` matches your n8n workflow
- For the Chrome extension, the webhook is hardcoded to `http://localhost:5678/webhook-test/cyborg-apply`

### Form fields not filling
- Some sites use React or Shadow DOM — the extension includes React-safe value setters
- The Playwright agent works best on standard Greenhouse and Lever forms
- For unsupported ATS platforms, the agent generates the PDF resume and returns `"Needs Human Input"`

### CAPTCHA blocking the agent
- The agent detects reCAPTCHA, Turnstile, and similar CAPTCHAs
- When detected, it stops and returns `"CAPTCHA"` status
- Solve the CAPTCHA manually in the open browser, then use `m` (manual submit) in the terminal

### Empty resume PDF
- Check the terminal output: `"WARNING: Resume text is EMPTY!"`
- Ensure your n8n workflow sends the `resumeMarkdown` field with content

---

## License

ISC
