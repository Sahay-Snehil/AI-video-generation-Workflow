# AI Content Generator — Setup Guide

## Overview

This n8n workflow automates AI-generated advertisement video and image creation. The client enters prompts into a Google Sheet; the workflow generates the content daily and returns download links.

---

## Quick Start

### Step 1: Set Up your Google Sheet

Create a Google Sheet with these **exact column headers** in Row 1:

| Row ID | Date | Content Type | Mandarin Script | Status | Download Link |
|--------|------|--------------|-----------------|--------|---------------|
| 1 | 2026-03-27 | video | 你好，我们提供专业的银行贷款服务... | pending | |
| 2 | 2026-03-27 | image | 信用卡优惠活动宣传图 | pending | |

- **Row ID**: Unique number for each row (1, 2, 3, ...)
- **Content Type**: Must be exactly `video` or `image`
- **Mandarin Script**: For videos, this is what the avatar will say. For images, this describes the image.
- **Status**: Set to `pending` for new entries. The workflow will change to `processing` → `completed` or `failed`.

### Step 2: Import the Workflow

1. Open your n8n instance
2. Go to **Workflows** → **Add Workflow** → **Import from File**
3. Select `AI_Content_Generator.json`
4. The workflow will load with all nodes visible

### Step 3: Configure Credentials

You need to set up **3 credentials** (email notification is optional):

#### 1. Google Sheets OAuth2
- In n8n, go to **Settings** → **Credentials** → **Add Credential**
- Search for **Google Sheets OAuth2 API**
- Follow the Google OAuth2 setup flow
- After creating, update **all 5 Google Sheets nodes** to use this credential
- In each Google Sheets node, select your actual Spreadsheet and Sheet name

#### 2. HeyGen API Key
- Sign up at [heygen.com](https://heygen.com)
- Go to **Settings** → **API** → Copy your API key
- In n8n, create an **HTTP Header Auth** credential:
  - **Name**: `X-Api-Key`
  - **Value**: Your HeyGen API key
- Update both HeyGen nodes to use this credential

#### 3. OpenAI API Key
- Get your key from [platform.openai.com](https://platform.openai.com)
- In n8n, create an **OpenAI API** credential with your key
- Update the "DALL-E 3 — Generate Image" node

#### 4. SMTP (Optional — for email notifications)
- Only needed if you want the workflow to email the client when content is ready
- Configure with your email provider's SMTP settings

### Step 4: Configure HeyGen Avatar & Voice

In the **"HeyGen — Generate Video"** node, update these values in the JSON body:

```json
"avatar_id": "YOUR_AVATAR_ID"    ← Pick a Malaysian Chinese avatar from HeyGen
"voice_id": "YOUR_MANDARIN_VOICE_ID"  ← Pick a Mandarin voice from HeyGen
```

**To find available avatars and voices:**
1. Log into HeyGen dashboard
2. Go to **Avatars** → Browse and note the ID of a suitable Malaysian Chinese avatar
3. Go to **Voices** → Filter by "Chinese (Mandarin)" and pick one with an accent closest to Malaysian

### Step 5: Activate the Workflow

1. Click the **Activate** toggle in the top-right corner
2. The workflow will now run **every day at 9:00 AM** automatically
3. It will process all rows where `Status = pending`

---

## How It Works

```
Daily Trigger (9 AM)
    ↓
Read Google Sheet (get all "pending" rows)
    ↓
Route: Video or Image?
    ├── VIDEO: Mark processing → HeyGen API → Wait → Poll status → Get download URL
    └── IMAGE: Mark processing → DALL-E 3 → Get download URL
    ↓
Write download link back to Google Sheet (status = "completed")
    ↓
(Optional) Email client with download link
```

---

## Node Map

| Node | What It Does |
|------|--------------|
| **Daily 9AM Trigger** | Starts the workflow every day at 9 AM |
| **Get Pending Prompts** | Reads all rows with `Status = pending` from Google Sheets |
| **Is Video or Image?** | Routes to video or image generation based on `Content Type` column |
| **Mark Processing** | Updates the row status to `processing` |
| **HeyGen — Generate Video** | Sends the Mandarin script to HeyGen API to create a talking-head video |
| **Wait 30s** | Waits 30 seconds for HeyGen to process |
| **HeyGen — Check Video Status** | Polls HeyGen API to see if the video is done |
| **Is Video Ready?** | If yes → extract URL. If no → loop back to Wait |
| **Extract Video Download URL** | Pulls the final MP4 download link from HeyGen response |
| **DALL-E 3 — Generate Image** | Generates a professional ad image with a Malaysian Chinese person |
| **Extract Image Download URL** | Pulls the generated image URL from OpenAI response |
| **Update Sheet — Completed** | Writes the download link back to Google Sheets, marks `completed` |
| **Update Sheet — Failed** | Marks the row as `failed` if something goes wrong |
| **Email Client — Content Ready** | Sends an optional email with the download link |

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| No rows processed | Check that `Status` column says exactly `pending` (case-sensitive) |
| HeyGen loops forever | Check your API key and avatar_id. Look at the HeyGen node output for error messages |
| DALL-E images look wrong | Refine the prompt in the "DALL-E 3 — Generate Image" node |
| Google Sheets error | Re-authenticate your Google Sheets credential and re-select the spreadsheet in each node |
| Email not sending | SMTP credential is optional — skip if not needed |

---

## Costs (Estimated)

| Service | Cost | Notes |
|---------|------|-------|
| HeyGen | ~$48/month (Creator plan) | ~15 minutes of video per month |
| OpenAI DALL-E 3 | ~$0.08/image (HD) | ~$2.    40/month for 30 images |
| n8n | Free (self-hosted) or $20/mo (cloud) | |
