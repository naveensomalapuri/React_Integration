# SmartApply — React Frontend Integration Guide

> **For the React / frontend developer.**
> This document covers every API endpoint, exact request and response shapes verified directly from the source code, the toggle-button pipeline flow, job card display logic, error handling patterns, and environment setup.
> The backend is already live. **React only talks to Django — never directly to FastAPI.**

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Base URLs & Environment Setup](#2-base-urls--environment-setup)
3. [Authentication](#3-authentication)
   - [3.1 Signup](#31-signup)
   - [3.2 Verify OTP (Activate Account)](#32-verify-otp-activate-account)
   - [3.3 Resend OTP](#33-resend-otp)
   - [3.4 Signin (Login)](#34-signin-login)
   - [3.5 Get Current User (me)](#35-get-current-user-me)
   - [3.6 Refresh Token](#36-refresh-token)
   - [3.7 Forgot Password](#37-forgot-password)
   - [3.8 Verify Reset OTP](#38-verify-reset-otp)
   - [3.9 Reset Password](#39-reset-password)
4. [Resume Profile Endpoints](#4-resume-profile-endpoints)
   - [4.1 Resume Header](#41-resume-header)
   - [4.2 Professional Summary](#42-professional-summary)
   - [4.3 Skills](#43-skills)
   - [4.4 Work Experience](#44-work-experience)
   - [4.5 Education](#45-education)
   - [4.6 Projects](#46-projects)
   - [4.7 Certifications](#47-certifications)
   - [4.8 Achievements](#48-achievements)
   - [4.9 Get Full Resume (Aggregate)](#49-get-full-resume-aggregate)
5. [SmartApply Pipeline — The Toggle Button](#5-smartapply-pipeline--the-toggle-button)
   - [5.1 How It Works](#51-how-it-works)
   - [5.2 Trigger the Full Pipeline](#52-trigger-the-full-pipeline)
   - [5.3 Full Response Shape](#53-full-response-shape)
   - [5.4 What to Display Per Job Card](#54-what-to-display-per-job-card)
   - [5.5 Resume Download](#55-resume-download)
6. [My Applications — Load Saved Results](#6-my-applications--load-saved-results)
7. [Discover Jobs Only (Agent 1)](#7-discover-jobs-only-agent-1)
8. [Agents Health Check](#8-agents-health-check)
9. [Error Handling](#9-error-handling)
10. [Auth UI Flow Diagrams](#10-auth-ui-flow-diagrams)
11. [Quick Reference — All Endpoints](#11-quick-reference--all-endpoints)
12. [Important Notes for the React Developer](#12-important-notes-for-the-react-developer)

---

## 1. Architecture Overview

```
React App
    │
    │  All requests use JWT Bearer token
    ▼
Django Backend (your only API contact)
https://smartapply-7msy.onrender.com
    │
    ├── /api/auth/*          → signup, login, OTP, password reset
    ├── /api/me/resume/*     → resume profile CRUD
    └── /api/agents/*        → triggers AI pipeline
                │
                │  Internal call using X-Service-Auth (you never see this)
                ▼
         FastAPI Agents Service
         https://smartapply-agents.onrender.com
                │
         Agent 1 → Searches company career pages for jobs matching user's target_role
         Agent 2 → Enriches jobs (full description, requirements, company info)
         Agent 3 → Scores each job against user skills & experience (0–100)
         Agent 4 → Generates a tailored PDF resume for high-scoring jobs (≥80)
                │
                └── Results returned back through Django → back to React
```

**Key point:** React only ever calls `https://smartapply-7msy.onrender.com`. Django handles all internal communication with the FastAPI agents service. You never need the FastAPI URL or the service auth key in your frontend code.

---

## 2. Base URLs & Environment Setup

### .env file

```env
# Vite
VITE_API_BASE_URL=https://smartapply-7msy.onrender.com

# Create React App
REACT_APP_API_BASE_URL=https://smartapply-7msy.onrender.com
```

### API Client Helper (`src/api/client.js`)

This helper automatically attaches the JWT token and handles token refresh on 401:

```js
const BASE = import.meta.env.VITE_API_BASE_URL;
// or: const BASE = process.env.REACT_APP_API_BASE_URL;

export async function apiFetch(path, options = {}) {
  const token = localStorage.getItem("access_token");

  const res = await fetch(`${BASE}${path}`, {
    ...options,
    headers: {
      "Content-Type": "application/json",
      ...(token ? { Authorization: `Bearer ${token}` } : {}),
      ...(options.headers || {}),
    },
  });

  if (res.status === 401) {
    const refreshed = await refreshAccessToken();
    if (!refreshed) {
      localStorage.removeItem("access_token");
      localStorage.removeItem("refresh_token");
      window.location.href = "/login";
      return;
    }
    return apiFetch(path, options); // retry once with new token
  }

  return res;
}

async function refreshAccessToken() {
  const refresh = localStorage.getItem("refresh_token");
  if (!refresh) return false;

  const res = await fetch(`${BASE}/api/auth/refresh/`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ refresh }),
  });

  if (!res.ok) return false;

  const data = await res.json();
  localStorage.setItem("access_token", data.access);
  return true;
}
```

---

## 3. Authentication

> **How the signup + login flow works:**
> 1. User submits signup form → OTP sent to their email
> 2. User enters 6-digit OTP → account activated + JWT tokens returned (user is now logged in)
> 3. For future logins: POST signin → JWT tokens returned
>
> Users **cannot sign in** until they complete OTP verification.

---

### 3.1 Signup

**`POST /api/auth/signup/`** — No authentication required.

**What it does:** Creates a new user account with `is_active=False`, generates a 6-digit OTP, and sends it to the user's email. The user cannot log in until they verify this OTP.

**Request body:**

| Field | Type | Required | Description |
|---|---|---|---|
| `email` | string | Yes | Must be unique. If email already has an inactive account, it reuses and updates it. |
| `password` | string | Yes | Minimum 8 characters. Django validates: not too common, not too short, not too similar to email. |
| `full_name` | string | Yes | User's display name. |

```js
const res = await apiFetch("/api/auth/signup/", {
  method: "POST",
  body: JSON.stringify({
    email: "user@example.com",
    password: "SecurePass123!",
    full_name: "John Doe",
  }),
});
const data = await res.json();
// On 200: navigate to OTP verification screen, passing the email
```

**Success — HTTP 200:**
```json
{
  "message": "OTP sent to user@example.com. Please verify to activate your account."
}
```

**Error — HTTP 400** (email already active):
```json
{
  "error": "A user with this email already exists."
}
```

**Error — HTTP 400** (missing or invalid fields):
```json
{
  "email": ["This field is required."],
  "full_name": ["This field is required."]
}
```

> After receiving HTTP 200, navigate to an OTP screen where the user enters the 6-digit code from their email. The OTP expires in **5 minutes**.

---

### 3.2 Verify OTP (Activate Account)

**`POST /api/auth/verify-otp/`** — No authentication required.

**What it does:** Verifies the 6-digit signup OTP. On success: marks the account as active, invalidates the OTP, and returns JWT tokens so the user is immediately logged in.

**Request body:**

| Field | Type | Required | Description |
|---|---|---|---|
| `email` | string | Yes | Same email used during signup. |
| `otp` | string | Yes | Exactly 6 digits. From the verification email. |

```js
const res = await apiFetch("/api/auth/verify-otp/", {
  method: "POST",
  body: JSON.stringify({
    email: "user@example.com",
    otp: "482910",
  }),
});
const data = await res.json();

if (res.ok) {
  // Store tokens — user is now logged in
  localStorage.setItem("access_token", data.access);
  localStorage.setItem("refresh_token", data.refresh);
  // Navigate to dashboard
}
```

**Success — HTTP 200:**
```json
{
  "access": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refresh": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "id": 1,
    "email": "user@example.com",
    "full_name": "John Doe"
  }
}
```

> Store `access` and `refresh` in `localStorage`. The `user.id` is the `user_id` you'll need elsewhere.

**Error — HTTP 400** (wrong OTP or expired):
```json
{
  "error": "Invalid or expired OTP."
}
```

**Error — HTTP 404** (email not found):
```json
{
  "error": "User not found."
}
```

> Rate limited to **10 OTP attempts per minute** to prevent brute-force attacks. On repeated failures, show a "Resend OTP" button.

---

### 3.3 Resend OTP

**`POST /api/auth/resend-otp/`** — No authentication required.

**What it does:** Deletes any existing unused OTP for this email, generates a new 6-digit OTP, and resends it. Only works for accounts that are still inactive (not yet verified).

**Request body:**

| Field | Type | Required | Description |
|---|---|---|---|
| `email` | string | Yes | Email of the unverified account. |

```js
const res = await apiFetch("/api/auth/resend-otp/", {
  method: "POST",
  body: JSON.stringify({
    email: "user@example.com",
  }),
});
const data = await res.json();
```

**Success — HTTP 200:**
```json
{
  "message": "OTP resent to user@example.com."
}
```

**Error — HTTP 404** (email not found):
```json
{
  "error": "User not found."
}
```

**Error — HTTP 400** (account already verified):
```json
{
  "error": "Account is already active."
}
```

> Show a "Resend OTP" button on the OTP screen. Recommend a 30-second cooldown UI to prevent spam clicking (the backend will resend regardless, but good UX practice).

---

### 3.4 Signin (Login)

**`POST /api/auth/signin/`** — No authentication required.

**What it does:** Authenticates an active user with email and password. Returns JWT access and refresh tokens. Only works for accounts that have completed OTP verification (`is_active=True`).

**Request body:**

| Field | Type | Required | Description |
|---|---|---|---|
| `email` | string | Yes | Registered and verified email address. |
| `password` | string | Yes | The account password. |

```js
const res = await apiFetch("/api/auth/signin/", {
  method: "POST",
  body: JSON.stringify({
    email: "user@example.com",
    password: "SecurePass123!",
  }),
});
const data = await res.json();

if (res.ok) {
  localStorage.setItem("access_token", data.access);
  localStorage.setItem("refresh_token", data.refresh);
  // Navigate to dashboard
}
```

**Success — HTTP 200:**
```json
{
  "access": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refresh": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

> **Token lifetimes:**
> - `access` token: **2 hours** — attach to every authenticated request as `Authorization: Bearer <access>`
> - `refresh` token: **7 days** — use to get a new access token when the old one expires

**Error — HTTP 401:**
```json
{
  "detail": "No active account found with the given credentials"
}
```

> This error covers both wrong password AND unverified accounts. If signup was just completed without OTP verification, tell the user to check their email.

---

### 3.5 Get Current User (me)

**`GET /api/auth/me/`** — Requires JWT.

**What it does:** Returns the authenticated user's basic profile. Use this on app load to verify the token is still valid and to get the user's `id`.

```js
const res = await apiFetch("/api/auth/me/");
const user = await res.json();
// user.id is needed for display and reference — but NOT required as input to other endpoints
// (Django reads user_id from the JWT token automatically)
```

**Success — HTTP 200:**
```json
{
  "id": 1,
  "email": "user@example.com",
  "full_name": "John Doe"
}
```

**Error — HTTP 401:** Token missing or invalid.

---

### 3.6 Refresh Token

**`POST /api/auth/refresh/`** — No authentication required (sends the refresh token in the body).

**What it does:** Exchanges a valid refresh token for a new access token. The `apiFetch` helper in Section 2 calls this automatically on 401 responses — you usually don't need to call this manually.

**Request body:**

| Field | Type | Required | Description |
|---|---|---|---|
| `refresh` | string | Yes | The refresh token stored in `localStorage`. |

```js
const res = await apiFetch("/api/auth/refresh/", {
  method: "POST",
  body: JSON.stringify({
    refresh: localStorage.getItem("refresh_token"),
  }),
});
const data = await res.json();
localStorage.setItem("access_token", data.access);
```

**Success — HTTP 200:**
```json
{
  "access": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

**Error — HTTP 401:** Refresh token expired or invalid — redirect to login.

---

### 3.7 Forgot Password

**`POST /api/auth/forgot-password/`** — No authentication required.

**What it does:** Sends a 6-digit OTP to the provided email for password reset. Always returns HTTP 200 regardless of whether the email exists — this prevents email enumeration (an attacker can't use this endpoint to check which emails are registered).

**Request body:**

| Field | Type | Required | Description |
|---|---|---|---|
| `email` | string | Yes | Any email address — always gets a 200 response. |

```js
const res = await apiFetch("/api/auth/forgot-password/", {
  method: "POST",
  body: JSON.stringify({
    email: "user@example.com",
  }),
});
const data = await res.json();
// Always HTTP 200 — navigate to OTP input screen regardless
```

**Success — HTTP 200** (always, even if email doesn't exist):
```json
{
  "message": "If an account with user@example.com exists, an OTP has been sent."
}
```

> After HTTP 200, navigate to an OTP input screen and tell the user to check their email. The OTP expires in **5 minutes**.

---

### 3.8 Verify Reset OTP

**`POST /api/auth/verify-reset-otp/`** — No authentication required.

**What it does:** Verifies the forgot-password OTP. On success, marks the OTP as used and returns a short-lived `reset_token` (a signed token valid for **10 minutes**) that is required in the next step. Only works for active accounts.

**Request body:**

| Field | Type | Required | Description |
|---|---|---|---|
| `email` | string | Yes | The email that received the password reset OTP. |
| `otp` | string | Yes | Exactly 6 digits. |

```js
const res = await apiFetch("/api/auth/verify-reset-otp/", {
  method: "POST",
  body: JSON.stringify({
    email: "user@example.com",
    otp: "193847",
  }),
});
const data = await res.json();

if (res.ok) {
  // Store in React state — NOT localStorage — it's single-use and expires in 10 min
  setResetToken(data.reset_token);
  // Navigate to "Set New Password" screen
}
```

**Success — HTTP 200:**
```json
{
  "reset_token": "ImV4YW1wbGVAZXhhbXBsZS5jb20i:1uAbCd:xyz123..."
}
```

**Error — HTTP 400** (wrong or expired OTP):
```json
{
  "error": "Invalid or expired OTP."
}
```

**Error — HTTP 400** (email not found or account inactive):
```json
{
  "error": "Invalid request."
}
```

> Rate limited to **10 attempts per minute**. Store `reset_token` in React state only — never in `localStorage`. It's a single-use Django signed token.

---

### 3.9 Reset Password

**`POST /api/auth/reset-password/`** — No authentication required.

**What it does:** Sets a new password for the user identified by the `reset_token`. After success, the user must log in again with the new password. The `reset_token` is consumed and can never be reused.

**Request body:**

| Field | Type | Required | Description |
|---|---|---|---|
| `reset_token` | string | Yes | The token returned by `verify-reset-otp`. |
| `new_password` | string | Yes | Minimum 8 characters. Validated by Django's password validators (not too common, not too short). |

```js
const res = await apiFetch("/api/auth/reset-password/", {
  method: "POST",
  body: JSON.stringify({
    reset_token: resetToken,        // from React state
    new_password: "NewSecure456!",
  }),
});
const data = await res.json();

if (res.ok) {
  setResetToken(null); // discard the token
  // Redirect to login page
}
```

**Success — HTTP 200:**
```json
{
  "message": "Password reset successful."
}
```

**Error — HTTP 400** (reset_token expired — older than 10 minutes):
```json
{
  "error": "Reset token has expired."
}
```

**Error — HTTP 400** (reset_token invalid or tampered):
```json
{
  "error": "Invalid reset token."
}
```

**Error — HTTP 400** (password too weak — returns a list):
```json
{
  "error": ["This password is too common.", "This password is too short."]
}
```

> After success, redirect to the login page. The user must sign in fresh with their new password.

---

## 4. Resume Profile Endpoints

All resume endpoints require `Authorization: Bearer <access_token>`.

The resume is split into 8 sections. The AI agents use the resume data to search for jobs and generate tailored resumes — **the more complete the profile, the better the job matches.**

---

### 4.1 Resume Header

One per user. Contains identity info and the critically important `target_role` that drives all job searching.

**`GET /api/me/resume/header/`** — Returns the header or `null` if not set yet.

**`PUT /api/me/resume/header/`** — Create or fully replace the header.

**`PATCH /api/me/resume/header/`** — Partially update specific fields.

**Request body for PUT (all fields):**

| Field | Type | Required | Description |
|---|---|---|---|
| `full_name` | string | Yes | Max 150 characters. |
| `target_role` | string | Yes | Max 150 characters. **Most important field** — AI agents search jobs based on this. E.g. "Senior React Native Developer", "Backend Python Engineer". |
| `phone` | string | Yes | Max 30 characters. E.g. "+91-9876543210". |
| `email` | string | Yes | Contact email (can differ from login email). |
| `location_city` | string | Yes | Max 120 characters. E.g. "Bangalore". |
| `location_country` | string | Yes | Max 120 characters. E.g. "India". |
| `linkedin_url` | string | No | Full URL. Optional. |
| `github_url` | string | No | Full URL. Optional. |
| `portfolio_url` | string | No | Full URL. Optional. |

```js
const res = await apiFetch("/api/me/resume/header/", {
  method: "PUT",
  body: JSON.stringify({
    full_name: "John Doe",
    target_role: "Senior React Native Developer",
    phone: "+91-9876543210",
    email: "john@example.com",
    location_city: "Bangalore",
    location_country: "India",
    linkedin_url: "https://linkedin.com/in/johndoe",
    github_url: "https://github.com/johndoe",
    portfolio_url: "",
  }),
});
const data = await res.json();
```

**Success — HTTP 200:**
```json
{
  "id": 1,
  "full_name": "John Doe",
  "target_role": "Senior React Native Developer",
  "phone": "+91-9876543210",
  "email": "john@example.com",
  "location_city": "Bangalore",
  "location_country": "India",
  "linkedin_url": "https://linkedin.com/in/johndoe",
  "github_url": "https://github.com/johndoe",
  "portfolio_url": ""
}
```

> **`target_role` drives everything.** If it's missing or empty, Agent 1 falls back to searching for "Software Engineer". Always prompt the user to fill this in before enabling the toggle.

---

### 4.2 Professional Summary

One per user. A 2–3 line text summary that appears at the top of generated resumes.

**`GET /api/me/resume/summary/`** — Returns the summary (auto-created with empty text if not set yet).

**`PUT /api/me/resume/summary/`** — Fully replace the summary.

**`PATCH /api/me/resume/summary/`** — Partial update.

**Request body:**

| Field | Type | Required | Description |
|---|---|---|---|
| `summary_text` | string | Yes (PUT) / No (PATCH) | Free-form text. No length limit in DB. Recommend 2–5 sentences. |

```js
// Save summary
await apiFetch("/api/me/resume/summary/", {
  method: "PUT",
  body: JSON.stringify({
    summary_text: "Experienced React Native developer with 3+ years building cross-platform mobile apps. Passionate about performance optimization and clean architecture.",
  }),
});

// Get summary
const res = await apiFetch("/api/me/resume/summary/");
const data = await res.json();
```

**Success — HTTP 200:**
```json
{
  "id": 1,
  "summary_text": "Experienced React Native developer with 3+ years building cross-platform mobile apps..."
}
```

---

### 4.3 Skills

Many per user. Each skill belongs to a category. The `(user, category, name)` combination must be unique — no duplicate skill names within the same category.

**`GET /api/me/resume/skills/`** — Returns all skills for the user, sorted by category then name.

**`POST /api/me/resume/skills/`** — Add a new skill.

**`PUT /api/me/resume/skills/{id}/`** — Fully update a skill.

**`PATCH /api/me/resume/skills/{id}/`** — Partially update a skill.

**`DELETE /api/me/resume/skills/{id}/`** — Delete a skill.

**Request body for POST/PUT:**

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | Yes | Skill name. Max 80 characters. E.g. "React Native", "Python", "PostgreSQL". |
| `category` | string | Yes | Must be one of the values below. |

**Valid `category` values:**

| Value | Display Label |
|---|---|
| `technical` | Technical Skills |
| `languages` | Programming Languages |
| `frameworks` | Frameworks & Libraries |
| `tools` | Tools & Platforms |
| `soft` | Soft Skills |

```js
// Add a skill
const res = await apiFetch("/api/me/resume/skills/", {
  method: "POST",
  body: JSON.stringify({
    name: "React Native",
    category: "frameworks",
  }),
});
const skill = await res.json();
// skill.id is used for update and delete

// List all skills
const listRes = await apiFetch("/api/me/resume/skills/");
const skills = await listRes.json();
// Returns: [{ id, name, category }, ...]

// Delete a skill
await apiFetch(`/api/me/resume/skills/${skill.id}/`, { method: "DELETE" });
```

**Success POST — HTTP 201:**
```json
{
  "id": 5,
  "name": "React Native",
  "category": "frameworks"
}
```

**Error — HTTP 400** (duplicate skill in same category):
```json
{
  "detail": "This entry already exists for your account."
}
```

**Error — HTTP 400** (invalid category):
```json
{
  "category": ["\"invalid_value\" is not a valid choice."]
}
```

---

### 4.4 Work Experience

Many per user. Each entry has bullet points describing achievements. Used by Agent 3 to calculate `experience_years` and by Agent 4 to generate resume content.

**`GET /api/me/resume/work-experience/`** — Returns all experiences, sorted by `ordering_index` then `-start_date`.

**`POST /api/me/resume/work-experience/`** — Add a new experience.

**`PUT /api/me/resume/work-experience/{id}/`** — Fully update.

**`PATCH /api/me/resume/work-experience/{id}/`** — Partially update.

**`DELETE /api/me/resume/work-experience/{id}/`** — Delete.

**Request body for POST/PUT:**

| Field | Type | Required | Description |
|---|---|---|---|
| `job_title` | string | Yes | Max 150 characters. E.g. "Senior Backend Developer". |
| `company_name` | string | Yes | Max 150 characters. E.g. "Google". |
| `start_date` | string | Yes | Format: `YYYY-MM-DD`. E.g. `"2022-03-01"`. |
| `end_date` | string or null | No | Format: `YYYY-MM-DD`. Set to `null` if this is the current role. |
| `bullets` | array of strings | No | List of resume bullet points describing achievements. Each must be a string. Default: `[]`. |
| `ordering_index` | number | No | Controls display order. Lower = shown first. Default: `0`. |

```js
// Add work experience
await apiFetch("/api/me/resume/work-experience/", {
  method: "POST",
  body: JSON.stringify({
    job_title: "Senior React Native Developer",
    company_name: "Acme Corp",
    start_date: "2022-01-01",
    end_date: null,           // null = currently working here
    bullets: [
      "Built cross-platform mobile app serving 500K active users",
      "Reduced app load time by 40% through lazy loading and caching",
      "Led a team of 4 developers across iOS and Android platforms",
    ],
    ordering_index: 0,
  }),
});
```

**Success — HTTP 201:**
```json
{
  "id": 3,
  "job_title": "Senior React Native Developer",
  "company_name": "Acme Corp",
  "start_date": "2022-01-01",
  "end_date": null,
  "bullets": [
    "Built cross-platform mobile app serving 500K active users",
    "Reduced app load time by 40%",
    "Led a team of 4 developers"
  ],
  "ordering_index": 0
}
```

**Error — HTTP 400** (bullets not a list):
```json
{
  "bullets": ["bullets must be a list of strings"]
}
```

---

### 4.5 Education

Many per user. Sorted by `ordering_index` then `-graduation_year` (newest first).

**`GET /api/me/resume/education/`** — List all.

**`POST /api/me/resume/education/`** — Add.

**`PUT /api/me/resume/education/{id}/`** — Update.

**`DELETE /api/me/resume/education/{id}/`** — Delete.

**Request body for POST/PUT:**

| Field | Type | Required | Description |
|---|---|---|---|
| `degree` | string | Yes | Max 200 characters. E.g. "B.Tech Computer Science", "MBA Finance". |
| `institution` | string | Yes | Max 200 characters. E.g. "IIT Bombay". |
| `graduation_year` | number | Yes | 4-digit year. E.g. `2021`. |
| `cgpa` | string | No | Stored as string to support formats like `"8.5/10"` or `"3.5/4"`. Max 20 chars. |
| `ordering_index` | number | No | Display order. Default: `0`. |

```js
await apiFetch("/api/me/resume/education/", {
  method: "POST",
  body: JSON.stringify({
    degree: "B.Tech Computer Science",
    institution: "IIT Bombay",
    graduation_year: 2021,
    cgpa: "8.7/10",
    ordering_index: 0,
  }),
});
```

**Success — HTTP 201:**
```json
{
  "id": 2,
  "degree": "B.Tech Computer Science",
  "institution": "IIT Bombay",
  "graduation_year": 2021,
  "cgpa": "8.7/10",
  "ordering_index": 0
}
```

---

### 4.6 Projects

Many per user. Sorted by `ordering_index` then `title`.

**`GET /api/me/resume/projects/`** — List all.

**`POST /api/me/resume/projects/`** — Add.

**`PUT /api/me/resume/projects/{id}/`** — Update.

**`DELETE /api/me/resume/projects/{id}/`** — Delete.

**Request body for POST/PUT:**

| Field | Type | Required | Description |
|---|---|---|---|
| `title` | string | Yes | Max 200 characters. |
| `tech_stack` | array of strings | No | List of technologies used. E.g. `["React Native", "Python", "FastAPI"]`. Must be array of strings. Default: `[]`. |
| `bullets` | array of strings | No | List of achievement/description bullet points. Must be array of strings. Default: `[]`. |
| `ordering_index` | number | No | Display order. Default: `0`. |

```js
await apiFetch("/api/me/resume/projects/", {
  method: "POST",
  body: JSON.stringify({
    title: "SmartApply — AI Job Application Automation",
    tech_stack: ["React Native", "Python", "FastAPI", "PostgreSQL", "Redis"],
    bullets: [
      "Built AI pipeline with 4 agents for automated job discovery and application",
      "Deployed on Render with Docker, serving 100+ users",
    ],
    ordering_index: 0,
  }),
});
```

**Success — HTTP 201:**
```json
{
  "id": 4,
  "title": "SmartApply — AI Job Application Automation",
  "tech_stack": ["React Native", "Python", "FastAPI", "PostgreSQL", "Redis"],
  "bullets": [
    "Built AI pipeline with 4 agents for automated job discovery",
    "Deployed on Render with Docker"
  ],
  "ordering_index": 0
}
```

**Error — HTTP 400** (tech_stack or bullets not arrays):
```json
{
  "tech_stack": ["tech_stack must be a list of strings"]
}
```

---

### 4.7 Certifications

Many per user. Sorted by `ordering_index`, then `-issue_year`, then `name`.

**`GET /api/me/resume/certifications/`** — List all.

**`POST /api/me/resume/certifications/`** — Add.

**`PUT /api/me/resume/certifications/{id}/`** — Update.

**`DELETE /api/me/resume/certifications/{id}/`** — Delete.

**Request body for POST/PUT:**

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | Yes | Max 200 characters. E.g. "AWS Certified Developer – Associate". |
| `organization` | string | No | Issuing organization. Max 200 characters. E.g. "Amazon Web Services". Note: field is `organization` not `issuer`. |
| `credential_url` | string | No | Full URL to verify the certificate. |
| `issue_year` | number | No | 4-digit year number — not a date string. E.g. `2023`. |
| `ordering_index` | number | No | Display order. Default: `0`. |

```js
await apiFetch("/api/me/resume/certifications/", {
  method: "POST",
  body: JSON.stringify({
    name: "AWS Certified Developer – Associate",
    organization: "Amazon Web Services",
    credential_url: "https://www.credly.com/badges/abc123",
    issue_year: 2023,
    ordering_index: 0,
  }),
});
```

**Success — HTTP 201:**
```json
{
  "id": 7,
  "name": "AWS Certified Developer – Associate",
  "organization": "Amazon Web Services",
  "credential_url": "https://www.credly.com/badges/abc123",
  "issue_year": 2023,
  "ordering_index": 0
}
```

> **Field names to watch:** `organization` (not `issuer`), `issue_year` (number, not `issue_date` string).

---

### 4.8 Achievements

Many per user. Sorted by `ordering_index`, then `-year`, then `title`.

**`GET /api/me/resume/achievements/`** — List all.

**`POST /api/me/resume/achievements/`** — Add.

**`PUT /api/me/resume/achievements/{id}/`** — Update.

**`DELETE /api/me/resume/achievements/{id}/`** — Delete.

**Request body for POST/PUT:**

| Field | Type | Required | Description |
|---|---|---|---|
| `title` | string | Yes | Max 200 characters. E.g. "1st Place — National Hackathon 2023". |
| `description` | string | No | Free text. Optional additional context. |
| `year` | number | No | 4-digit year. E.g. `2023`. |
| `type` | string | No | One of the values below. Default: `"other"`. |
| `ordering_index` | number | No | Display order. Default: `0`. |

**Valid `type` values:**

| Value | Label |
|---|---|
| `award` | Award |
| `hackathon` | Hackathon |
| `publication` | Publication |
| `other` | Other |

```js
await apiFetch("/api/me/resume/achievements/", {
  method: "POST",
  body: JSON.stringify({
    title: "1st Place — Internal Hackathon 2023",
    description: "Won company-wide 48-hour hackathon with an AI-powered productivity tool",
    year: 2023,
    type: "hackathon",
    ordering_index: 0,
  }),
});
```

**Success — HTTP 201:**
```json
{
  "id": 9,
  "title": "1st Place — Internal Hackathon 2023",
  "description": "Won company-wide 48-hour hackathon",
  "year": 2023,
  "type": "hackathon",
  "ordering_index": 0
}
```

---

### 4.9 Get Full Resume (Aggregate)

Returns the entire resume in a single API call. Use this for a "View Resume" page or to pre-load the profile editor.

**`GET /api/me/resume/`** — Requires JWT.

```js
const res = await apiFetch("/api/me/resume/");
const resume = await res.json();
```

**Success — HTTP 200:**
```json
{
  "header": {
    "id": 1,
    "full_name": "John Doe",
    "target_role": "Senior React Native Developer",
    "phone": "+91-9876543210",
    "email": "john@example.com",
    "location_city": "Bangalore",
    "location_country": "India",
    "linkedin_url": "https://linkedin.com/in/johndoe",
    "github_url": "https://github.com/johndoe",
    "portfolio_url": ""
  },
  "summary": {
    "id": 1,
    "summary_text": "Experienced React Native developer with 3+ years..."
  },
  "skills": [
    { "id": 1, "name": "React Native", "category": "frameworks" },
    { "id": 2, "name": "TypeScript", "category": "languages" },
    { "id": 3, "name": "Python", "category": "languages" }
  ],
  "work_experience": [
    {
      "id": 3,
      "job_title": "Senior React Native Developer",
      "company_name": "Acme Corp",
      "start_date": "2022-01-01",
      "end_date": null,
      "bullets": ["Built app serving 500K users", "Reduced load time 40%"],
      "ordering_index": 0
    }
  ],
  "projects": [
    {
      "id": 4,
      "title": "SmartApply",
      "tech_stack": ["React Native", "Python", "FastAPI"],
      "bullets": ["Built AI pipeline with 4 agents"],
      "ordering_index": 0
    }
  ],
  "education": [
    {
      "id": 2,
      "degree": "B.Tech Computer Science",
      "institution": "IIT Bombay",
      "graduation_year": 2021,
      "cgpa": "8.7/10",
      "ordering_index": 0
    }
  ],
  "certifications": [
    {
      "id": 7,
      "name": "AWS Certified Developer",
      "organization": "Amazon Web Services",
      "credential_url": "https://credly.com/badges/abc123",
      "issue_year": 2023,
      "ordering_index": 0
    }
  ],
  "achievements": [
    {
      "id": 9,
      "title": "1st Place — Hackathon 2023",
      "description": "Won company hackathon",
      "year": 2023,
      "type": "hackathon",
      "ordering_index": 0
    }
  ]
}
```

> `header` and `summary` can be `null` if the user hasn't filled them in yet. All array fields (`skills`, `work_experience`, etc.) return empty arrays `[]` if nothing has been added.

---

## 5. SmartApply Pipeline — The Toggle Button

### 5.1 How It Works

```
User enables the "SmartApply" toggle
           │
           ▼
  POST /api/agents/smart-apply/
           │
           │  Django fetches user profile from DB
           │  Django calls FastAPI with service auth key (internal)
           │
           ▼  Agent 1: Discovers jobs from company career pages
           ▼  Agent 2: Enriches each job (full description, requirements, salary)
           ▼  Agent 3: Scores each job 0–100 against user's skills & experience
           ▼  Agent 4: Generates tailored PDF resume for jobs scored ≥ 80
           │
           │  (Wait 30 seconds – 3 minutes, show loading UI)
           │
           ▼
  Response: jobs_to_apply (score ≥ 80) + jobs_to_notify (score 60–79)
           │
           ▼
  Display job cards with score, decision, and resume download button
```

**Score thresholds:**
- `score ≥ 80` → `decision: "auto_apply"` → resume is generated, shown in `jobs_to_apply`
- `score 60–79` → `decision: "notify"` → no resume, shown in `jobs_to_notify`
- `score < 60` → `decision: "rejected"` → never returned in the response

---

### 5.2 Trigger the Full Pipeline

**`POST /api/agents/smart-apply/`** — Requires JWT. Set a **5-minute timeout**.

**Request body:**

| Field | Type | Required | Description |
|---|---|---|---|
| `force_refresh` | boolean | No | Default `false`. When `false`, reuses the cached company list (7-day cache) — faster. When `true`, re-discovers companies from scratch — use after user updates `target_role`. |

```js
async function runSmartApply(forceRefresh = false) {
  const res = await fetch(`${BASE}/api/agents/smart-apply/`, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      Authorization: `Bearer ${localStorage.getItem("access_token")}`,
    },
    body: JSON.stringify({
      force_refresh: forceRefresh,
    }),
    signal: AbortSignal.timeout(300_000), // 5 minute timeout — do NOT use less
  });

  if (!res.ok) {
    const err = await res.json();
    throw new Error(err.error || "Pipeline failed");
  }

  return res.json();
}
```

**When the service is down — HTTP 503:**
```json
{
  "success": false,
  "error": "Failed to connect to agents service. Please try again."
}
```

**Internal error — HTTP 500:**
```json
{
  "success": false,
  "error": "..."
}
```

---

### 5.3 Full Response Shape

**Success — HTTP 200:**

```json
{
  "success": true,
  "user_id": "1",
  "total_jobs_found": 12,

  "jobs_to_apply": [
    {
      "job": {
        "job_id": "aa01297235265e0a248523e5956a7a46",
        "title": "Senior React Native Engineer",
        "company": "Stripe",
        "location": "Remote, USA",
        "source_url": "https://stripe.com/jobs/listing/senior-react-native",
        "source_platform": "stripe.com",
        "description": "We are looking for a Senior React Native Engineer to join our team...",
        "requirements": [
          "5+ years React Native experience",
          "TypeScript proficiency",
          "Experience with REST APIs and GraphQL"
        ],
        "nice_to_have": [
          "GraphQL experience",
          "iOS/Android native development background"
        ],
        "company_info": {
          "name": "Stripe",
          "domain": "stripe.com",
          "size": null,
          "industry": null,
          "description": null
        },
        "apply_link": "https://stripe.com/jobs/listing/senior-react-native/apply",
        "salary_info": {
          "min_salary": 150000,
          "max_salary": 200000,
          "currency": "USD",
          "period": "year"
        },
        "job_type": "full_time",
        "remote_type": "remote",
        "is_enriched": true
      },
      "score": 87.5,
      "decision": "auto_apply",
      "resume_url": "/app/generated_resumes/resume_1_aa012972_20260405.pdf",
      "apply_link": "https://stripe.com/jobs/listing/senior-react-native/apply"
    }
  ],

  "jobs_to_notify": [
    {
      "job": {
        "job_id": "0b7c68862be97dd9efd085f571569387",
        "title": "Senior Python Developer",
        "company": "Apple",
        "location": null,
        "source_url": "https://jobs.apple.com/en-us/details/200622263",
        "source_platform": "jobs.apple.com",
        "description": "We are seeking a Senior Python Developer...",
        "requirements": ["Python", "Django", "AWS"],
        "nice_to_have": [],
        "company_info": {
          "name": "Apple",
          "domain": "apple.com",
          "size": null,
          "industry": null,
          "description": null
        },
        "apply_link": null,
        "salary_info": null,
        "job_type": null,
        "remote_type": null,
        "is_enriched": true
      },
      "score": 71.0,
      "decision": "notify",
      "resume_url": null,
      "apply_link": null
    }
  ],

  "execution_time_ms": 45231,

  "agent_logs": [
    {
      "agent": "Agent 1: Discovery",
      "status": "success",
      "execution_time_ms": 12000,
      "details": "Found 8 new jobs"
    },
    {
      "agent": "Agent 2: Enrichment",
      "status": "success",
      "execution_time_ms": 18000,
      "details": "Enriched 8 jobs"
    },
    {
      "agent": "Agent 3: Scoring",
      "status": "success",
      "execution_time_ms": 3000,
      "details": "Scored 8 jobs"
    },
    {
      "agent": "Agent 4: Resume Generation",
      "status": "completed",
      "execution_time_ms": 12000,
      "details": "Generated 2 resumes; 0 escalated to notify"
    }
  ]
}
```

**Field notes:**
- `total_jobs_found` — total jobs discovered by Agent 1 (includes cached + new)
- `jobs_to_apply` — jobs with `score ≥ 80`, resume was generated
- `jobs_to_notify` — jobs with `score 60–79`, no resume
- `resume_url` — only non-null on `jobs_to_apply` items (see Section 5.5)
- `apply_link` — use this to open the job application. Can be `null` — fall back to `job.source_url`
- `agent_logs` — useful for showing progress details or debugging failed runs

---

### 5.4 What to Display Per Job Card

Every item in `jobs_to_apply` and `jobs_to_notify` has the same shape. Here's exactly which field maps to which UI element:

| UI Element | Field Path | Notes |
|---|---|---|
| Job title | `job.title` | Always present |
| Company name | `job.company` | May contain page noise like "Jobs at Apple" — display as-is |
| Location | `job.location` | Can be `null` — show "Not specified" |
| Score badge | `score` | Number 0–100, one decimal place |
| Decision badge | `decision` | `"auto_apply"` or `"notify"` |
| Job description | `job.description` | Full text string, can be long — use a truncate/expand pattern |
| Requirements | `job.requirements` | Array of strings. Can be empty `[]` |
| Nice to have | `job.nice_to_have` | Array of strings. Can be empty `[]` |
| Salary | `job.salary_info` | Object or `null` — see formatter below |
| Job type | `job.job_type` | `"full_time"`, `"part_time"`, `"contract"`, `"internship"`, or `null` |
| Remote type | `job.remote_type` | `"remote"`, `"hybrid"`, `"on_site"`, or `null` |
| Apply button | `apply_link` then `job.source_url` | Use `apply_link` first; if null fall back to `source_url` |
| Resume badge | `resume_url` | Non-null only on `auto_apply` jobs |
| Company domain | `job.company_info.domain` | Can be `null` |
| Platform | `job.source_platform` | E.g. "stripe.com", "jobs.apple.com" |

**Score badge color:**

```js
function scoreColor(score) {
  if (score >= 80) return "green";   // auto_apply — resume generated
  if (score >= 60) return "yellow";  // notify — good match
  return "red";                       // rejected — never returned
}
```

**Decision badge label:**

```js
function decisionLabel(decision) {
  if (decision === "auto_apply") return "Resume Generated ✓";
  if (decision === "notify")     return "Good Match";
  return "";
}
```

**Job type display:**

```js
function jobTypeLabel(jobType) {
  const labels = {
    full_time:   "Full Time",
    part_time:   "Part Time",
    contract:    "Contract",
    internship:  "Internship",
  };
  return jobType ? labels[jobType] : null;
}
```

**Remote type display:**

```js
function remoteTypeLabel(remoteType) {
  const labels = {
    remote:  "Remote",
    hybrid:  "Hybrid",
    on_site: "On-Site",
  };
  return remoteType ? labels[remoteType] : null;
}
```

**Salary display:**

```js
function formatSalary(salaryInfo) {
  if (!salaryInfo) return null;
  const { min_salary, max_salary, currency, period } = salaryInfo;
  if (min_salary && max_salary) {
    return `${currency} ${min_salary.toLocaleString()} – ${max_salary.toLocaleString()} / ${period}`;
  }
  if (min_salary) return `${currency} ${min_salary.toLocaleString()}+ / ${period}`;
  return null;
}
// Example output: "USD 150,000 – 200,000 / year"
```

**Apply button:**

```js
function getApplyUrl(jobResult) {
  return jobResult.apply_link || jobResult.job.source_url;
}
// Always open in a new tab: window.open(getApplyUrl(job), "_blank")
```

---

### 5.5 Resume Download

The `resume_url` field on `auto_apply` jobs is an internal server file path like:
```
/app/generated_resumes/resume_1_aa012972_20260405.pdf
```

> **Important:** This is a file path on the FastAPI container's filesystem, not a public URL. There is currently no public download endpoint for resume files.
>
> **What to do in the UI:**
> - If `resume_url` is not `null` → show a **"Resume Ready"** badge or indicator
> - The actual download button can be wired up once a download endpoint is added (backend work needed)
>
> **When the download endpoint is ready**, it will likely be:
> `GET /api/agents/resume/{job_id}/` — this will proxy the file from FastAPI back through Django

For now: show "Resume Generated ✓" as a badge. Do not attempt to fetch the `resume_url` path directly from the browser.

---

## 6. My Applications — Load Saved Results

Use this on dashboard load to show the user's previous pipeline results **without re-running the pipeline**. The pipeline can take 1-3 minutes so you should always check this first.

**`GET /api/agents/my-applications/`** — Requires JWT.

**Query parameters:**

| Param | Type | Default | Description |
|---|---|---|---|
| `skip` | number | `0` | Offset for pagination. |
| `limit` | number | `50` | Max records to return. |

```js
const res = await apiFetch("/api/agents/my-applications/?skip=0&limit=50");
const data = await res.json();
```

**Success — HTTP 200:**
```json
{
  "user_id": "1",
  "count": 18,
  "applications": [
    {
      "job_id": "aa01297235265e0a248523e5956a7a46",
      "fit_score": 87.5,
      "decision": "auto_apply",
      "resume_generated": true,
      "applied": false,
      "created_at": "2026-04-05T04:31:18.783071"
    },
    {
      "job_id": "0b7c68862be97dd9efd085f571569387",
      "fit_score": 71.0,
      "decision": "notify",
      "resume_generated": false,
      "applied": false,
      "created_at": "2026-04-05T04:31:20.123456"
    }
  ]
}
```

**Field notes:**
- `count` — total number of applications for this user
- `fit_score` — the AI score (0–100) assigned during the pipeline run
- `decision` — `"auto_apply"` (≥80) or `"notify"` (60–79)
- `resume_generated` — `true` if Agent 4 produced a PDF for this job
- `applied` — `false` by default (manual application tracking not yet implemented)
- `created_at` — ISO 8601 timestamp of when this application was saved

> This endpoint returns **summary data only** — score, decision, timestamps. It does not return full job details (title, company, description). For full details use the pipeline response from Section 5 and cache it in React state or localStorage.

**Recommended pattern:**

```js
async function loadDashboard() {
  // 1. Check for cached pipeline results in localStorage
  const cached = localStorage.getItem("last_pipeline_result");
  if (cached) {
    setJobResults(JSON.parse(cached));
  }

  // 2. Load application summary from the API (fast)
  const res = await apiFetch("/api/agents/my-applications/");
  const data = await res.json();
  setApplicationSummary(data.applications);
}
```

---

## 7. Discover Jobs Only (Agent 1)

Run only Agent 1 (job discovery) without scoring or resume generation. Useful if you want to show the user how many jobs were found before running the full pipeline.

**`POST /api/agents/discover-jobs/`** — Requires JWT.

**Request body:**

| Field | Type | Required | Description |
|---|---|---|---|
| `force_refresh` | boolean | No | Default `false`. |

```js
const res = await apiFetch("/api/agents/discover-jobs/", {
  method: "POST",
  body: JSON.stringify({ force_refresh: false }),
});
const data = await res.json();
```

**Success — HTTP 200:**
```json
{
  "success": true,
  "user_id": "1",
  "total_found": 12,
  "new_jobs": 8,
  "duplicate_jobs": 4,
  "jobs": [
    {
      "job_id": "aa012972...",
      "title": "Senior React Native Engineer",
      "company": "Stripe",
      "location": null,
      "source_url": "https://stripe.com/jobs/listing/123",
      "source_platform": "stripe.com"
    }
  ],
  "execution_time_ms": 12000
}
```

**Field notes:**
- `total_found` — total jobs found (new + deduplicated from cache)
- `new_jobs` — jobs not seen before in the last 30 days
- `duplicate_jobs` — jobs already in the database from previous runs
- `jobs` — basic listing (no description/requirements — those come from Agent 2)

---

## 8. Agents Health Check

Check if the FastAPI agents service is reachable before enabling the toggle. Call this when the dashboard loads.

**`GET /api/agents/health/`** — Requires JWT.

```js
const res = await apiFetch("/api/agents/health/");
const health = await res.json();

if (!health.can_process_jobs) {
  // Disable the SmartApply toggle and show a warning
  setToggleDisabled(true);
  showWarning("Job search service is temporarily unavailable.");
}
```

**Success — HTTP 200 (service is up):**
```json
{
  "fastapi_service": "healthy",
  "can_process_jobs": true
}
```

**Service down — HTTP 200 (but can_process_jobs is false):**
```json
{
  "fastapi_service": "unhealthy",
  "can_process_jobs": false
}
```

---

## 9. Error Handling

### HTTP Status Codes

| Code | Meaning | What to do in the UI |
|---|---|---|
| `200` / `201` | Success | Proceed normally |
| `400` | Validation error | Show the specific field errors from the response body |
| `401` | Unauthenticated / Token expired | Try token refresh (handled by `apiFetch`). If refresh fails → redirect to login |
| `403` | Forbidden | Show "You don't have permission" message |
| `404` | Not found | Resource doesn't exist (e.g. wrong OTP email) |
| `429` | Too many requests | OTP endpoints are rate-limited to 10/min — show "Too many attempts, wait a minute" |
| `500` | Server error | Show generic error, offer a retry button |
| `503` | Service unavailable | FastAPI agents service is down — show "Job search is temporarily unavailable" |

### Pipeline Error Handling

```js
async function handleToggle(enabled) {
  setAgentsEnabled(enabled);
  if (!enabled) return;

  setPipelineStatus("running");

  try {
    const result = await runSmartApply(false);

    if (!result.success) {
      showError("Pipeline failed. Please try again.");
      setPipelineStatus("error");
      setAgentsEnabled(false);
      return;
    }

    // Check for partial failures in agent logs
    const failedAgents = result.agent_logs.filter(log => log.status === "failed");
    if (failedAgents.length > 0) {
      showWarning(`Some steps had issues: ${failedAgents.map(a => a.agent).join(", ")}`);
    }

    // Cache results so user can navigate without re-running
    localStorage.setItem("last_pipeline_result", JSON.stringify(result));

    setJobResults(result);
    setPipelineStatus("done");

  } catch (err) {
    if (err.name === "TimeoutError") {
      showError("The search is taking longer than expected. Please try again in a few minutes.");
    } else {
      showError("Something went wrong. Please try again.");
    }
    setPipelineStatus("error");
    setAgentsEnabled(false);
  }
}
```

### Validation Error Display

Django returns field-level errors in different formats depending on the view:

```js
async function handleSignup(formData) {
  const res = await apiFetch("/api/auth/signup/", {
    method: "POST",
    body: JSON.stringify(formData),
  });
  const data = await res.json();

  if (!res.ok) {
    // Django REST serializer errors: { "field_name": ["error message"] }
    // Django view errors: { "error": "message" }
    if (data.error) {
      showError(Array.isArray(data.error) ? data.error.join(", ") : data.error);
    } else {
      // Field-level errors from serializer
      Object.entries(data).forEach(([field, errors]) => {
        setFieldError(field, errors[0]);
      });
    }
  }
}
```

---

## 10. Auth UI Flow Diagrams

### Signup Flow

```
┌──────────────────────────────────┐
│  Create Account                   │
│                                   │
│  Full Name  [__________________]  │
│  Email      [__________________]  │
│  Password   [__________________]  │
│                                   │
│  [Sign Up]                        │
└──────────────────────────────────┘
         │ POST /api/auth/signup/
         │ HTTP 200
         ▼
┌──────────────────────────────────┐
│  Verify Your Email                │
│                                   │
│  We sent a 6-digit code to        │
│  user@example.com                 │
│                                   │
│  [_] [_] [_] [_] [_] [_]         │
│                                   │
│  [Verify]                         │
│                                   │
│  Didn't receive it?               │
│  [Resend Code] (30s cooldown)     │
└──────────────────────────────────┘
         │ POST /api/auth/verify-otp/
         │ HTTP 200 → tokens returned
         ▼
┌──────────────────────────────────┐
│  Dashboard (logged in)            │
└──────────────────────────────────┘
```

### Forgot Password Flow

```
┌──────────────────────────────────┐
│  Forgot Password                  │
│                                   │
│  Email  [______________________]  │
│                                   │
│  [Send Reset Code]                │
└──────────────────────────────────┘
         │ POST /api/auth/forgot-password/
         │ HTTP 200 (always)
         ▼
┌──────────────────────────────────┐
│  Enter Reset Code                 │
│                                   │
│  [_] [_] [_] [_] [_] [_]         │
│                                   │
│  [Verify Code]                    │
└──────────────────────────────────┘
         │ POST /api/auth/verify-reset-otp/
         │ HTTP 200 → reset_token in state
         ▼
┌──────────────────────────────────┐
│  Set New Password                 │
│                                   │
│  New Password   [______________]  │
│  Confirm        [______________]  │
│                                   │
│  [Reset Password]                 │
└──────────────────────────────────┘
         │ POST /api/auth/reset-password/
         │ HTTP 200
         ▼
┌──────────────────────────────────┐
│  Password reset! [Sign In →]      │
└──────────────────────────────────┘
```

### SmartApply Dashboard

```
┌──────────────────────────────────────────────┐
│  SmartApply                                   │
│                                              │
│  Auto Job Search   [  OFF ──── ON toggle  ]  │
│                                              │
│  ── When toggle is OFF ──────────────────    │
│  "Enable to automatically find and score     │
│   jobs matching your target role."           │
│                                              │
│  ── When toggle is turned ON ────────────    │
│  ┌──────────────────────────────────────┐   │
│  │  Searching company career pages...    │   │
│  │  Agent 1  ████████░░░░  Discovering  │   │
│  │  Agent 2  ░░░░░░░░░░░░  Waiting...   │   │
│  │  Agent 3  ░░░░░░░░░░░░  Waiting...   │   │
│  │  Agent 4  ░░░░░░░░░░░░  Waiting...   │   │
│  └──────────────────────────────────────┘   │
│                                              │
│  ── When results return ─────────────────    │
│                                              │
│  Found 12 jobs • 2 resumes generated         │
│                                              │
│  ┌───────────────────────────────────────┐  │
│  │ Senior React Native Engineer     87 ● │  │
│  │ Stripe • Remote • Full Time           │  │
│  │ USD 150,000 – 200,000 / year          │  │
│  │ ✓ Resume Generated                    │  │
│  │ [View Job ↗]  [Resume Ready]          │  │
│  └───────────────────────────────────────┘  │
│                                              │
│  ┌───────────────────────────────────────┐  │
│  │ Senior Python Developer          71 ◐ │  │
│  │ Apple • Location not specified        │  │
│  │ Good Match                            │  │
│  │ [View Job ↗]                          │  │
│  └───────────────────────────────────────┘  │
└──────────────────────────────────────────────┘
```

### Recommended React State Shape

```js
const [agentsEnabled, setAgentsEnabled] = useState(false);
const [pipelineStatus, setPipelineStatus] = useState("idle");
// "idle" | "running" | "done" | "error"

const [jobResults, setJobResults] = useState({
  jobs_to_apply: [],
  jobs_to_notify: [],
  total_jobs_found: 0,
  agent_logs: [],
  execution_time_ms: 0,
});

// On dashboard mount — load previous results without re-running
useEffect(() => {
  const cached = localStorage.getItem("last_pipeline_result");
  if (cached) {
    setJobResults(JSON.parse(cached));
    setPipelineStatus("done");
  }
}, []);
```

---

## 11. Quick Reference — All Endpoints

All endpoints are on `https://smartapply-7msy.onrender.com`

| Method | Path | Auth | What it does |
|---|---|---|---|
| `POST` | `/api/auth/signup/` | None | Register → sends OTP to email |
| `POST` | `/api/auth/verify-otp/` | None | Verify signup OTP → activate account + get JWT |
| `POST` | `/api/auth/resend-otp/` | None | Resend signup OTP (for inactive accounts) |
| `POST` | `/api/auth/signin/` | None | Login with email+password → get JWT |
| `GET` | `/api/auth/me/` | JWT | Get current user's id, email, full_name |
| `POST` | `/api/auth/refresh/` | None | Exchange refresh token for new access token |
| `POST` | `/api/auth/forgot-password/` | None | Send forgot-password OTP to email |
| `POST` | `/api/auth/verify-reset-otp/` | None | Verify OTP → get reset_token (10 min expiry) |
| `POST` | `/api/auth/reset-password/` | None | Set new password using reset_token |
| `GET` | `/api/me/resume/` | JWT | Get full resume (all 8 sections in one call) |
| `GET/PUT/PATCH` | `/api/me/resume/header/` | JWT | Resume header (name, target_role, contact info) |
| `GET/PUT/PATCH` | `/api/me/resume/summary/` | JWT | Professional summary text |
| `GET/POST` | `/api/me/resume/skills/` | JWT | List all skills / add new skill |
| `GET/PUT/PATCH/DELETE` | `/api/me/resume/skills/{id}/` | JWT | Read / update / delete a skill |
| `GET/POST` | `/api/me/resume/work-experience/` | JWT | List all experience / add new |
| `GET/PUT/PATCH/DELETE` | `/api/me/resume/work-experience/{id}/` | JWT | Read / update / delete an experience |
| `GET/POST` | `/api/me/resume/education/` | JWT | List all education / add new |
| `GET/PUT/PATCH/DELETE` | `/api/me/resume/education/{id}/` | JWT | Read / update / delete education |
| `GET/POST` | `/api/me/resume/projects/` | JWT | List all projects / add new |
| `GET/PUT/PATCH/DELETE` | `/api/me/resume/projects/{id}/` | JWT | Read / update / delete a project |
| `GET/POST` | `/api/me/resume/certifications/` | JWT | List all certifications / add new |
| `GET/PUT/PATCH/DELETE` | `/api/me/resume/certifications/{id}/` | JWT | Read / update / delete a certification |
| `GET/POST` | `/api/me/resume/achievements/` | JWT | List all achievements / add new |
| `GET/PUT/PATCH/DELETE` | `/api/me/resume/achievements/{id}/` | JWT | Read / update / delete an achievement |
| `POST` | `/api/agents/smart-apply/` | JWT | **Run full pipeline** (all 4 agents) |
| `POST` | `/api/agents/discover-jobs/` | JWT | Run only Agent 1 (job discovery) |
| `GET` | `/api/agents/my-applications/` | JWT | Get all saved job application summaries |
| `GET` | `/api/agents/health/` | JWT | Check if agents service is reachable |

---

## 12. Important Notes for the React Developer

1. **React only talks to Django** — the URL `https://smartapply-7msy.onrender.com` is the only base URL you need. Never call FastAPI directly from the frontend.

2. **JWT is required on every resume and agent endpoint** — attach `Authorization: Bearer <access_token>` to all calls except the 9 auth endpoints (signup, verify-otp, resend-otp, signin, refresh, forgot-password, verify-reset-otp, reset-password).

3. **Access token expires in 2 hours, refresh token in 7 days** — the `apiFetch` helper handles this automatically. On any 401, it calls `/api/auth/refresh/` once and retries. If refresh fails, it clears localStorage and redirects to `/login`.

4. **Signup is a 2-step flow** — after `POST /api/auth/signup/` returns 200, you do NOT have tokens yet. Tokens are returned only after `POST /api/auth/verify-otp/` succeeds. Navigate to the OTP screen and wait for the user to enter the code.

5. **OTP expires in 5 minutes** — show a "Resend OTP" button on the OTP screen. Consider a visible 5-minute countdown timer to reduce support requests.

6. **reset_token is single-use, 10-minute expiry, store in state only** — never put it in localStorage. Once `reset-password` succeeds, discard it from state immediately.

7. **`target_role` drives the entire AI pipeline** — if the user hasn't set it in their Resume Header, Agent 1 falls back to searching "Software Engineer". Always prompt the user to fill in `target_role` before they enable the SmartApply toggle. Consider a "Profile incomplete" banner.

8. **The pipeline can take 30 seconds to 3 minutes** — set your fetch timeout to **5 minutes** (`AbortSignal.timeout(300_000)`). Never use the browser's default 30-second timeout. Show a loading state with agent progress indicators.

9. **`force_refresh: false` for normal runs** — the backend caches the list of companies found for 7 days. Normal runs reuse this cache (much faster). Only use `force_refresh: true` when the user updates their `target_role` or explicitly requests a fresh search.

10. **Cache pipeline results in localStorage** — the pipeline is expensive and takes time. After a successful run, store the result so the user can browse job cards without triggering another run when they navigate or refresh the page.

11. **`jobs_to_apply` are the priority items** — these have `score ≥ 80` and a generated resume. Always show them above `jobs_to_notify` in the UI. Use a tab or section separator: "Ready to Apply" and "Good Matches".

12. **`resume_url` is a server-side file path, not a public URL** — you cannot link to it directly. Show "Resume Ready" as a badge. The download button should be disabled / hidden until a download endpoint is added to the backend.

13. **Render free tier cold starts** — both services may take 30–60 seconds to respond after being idle. On the first request of the day, show a "Waking up service..." message if response takes more than 10 seconds.
