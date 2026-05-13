# BDR Onboarding Academy — Design Spec

**Date:** 2026-05-13  
**Author:** Sankhya Bantalpad (Product Marketing / Enablement)  
**Platform:** Next.js, hosted on Vercel  
**Goal:** Get new BDRs to generate 2 SQLs in month 1 and 2 SQLs in month 2

---

## Overview

The BDR Onboarding Academy is a self-directed learning platform with 7 courses (session deck, recording, quiz). This spec covers the new features being added on top of the existing content: BDR enrollment, sequential locking, completion and quiz tracking, automated nudges, HubSpot SQL tracking, an admin dashboard, and course feedback.

---

## Tech Stack

| Layer | Tool |
|---|---|
| Framework | Next.js (existing, on Vercel) |
| Database | Vercel Postgres or Supabase (Postgres) |
| Scheduled jobs | Vercel Cron |
| Email delivery | Resend (or SendGrid) |
| CRM integration | HubSpot API |

---

## Data Model

### `bdrs`
| Field | Type | Notes |
|---|---|---|
| id | uuid | Primary key |
| hubspot_contact_id | text | HubSpot contact ID, set at enrollment |
| name | text | Full name, pulled from HubSpot at enrollment |
| email | text | Pulled from HubSpot at enrollment, used for nudge emails |
| onboarding_start_date | date | Set at enrollment by admin |

### `course_progress`
| Field | Type | Notes |
|---|---|---|
| id | uuid | Primary key |
| bdr_id | uuid | FK → bdrs |
| course_id | int | 1–7 |
| started_at | timestamp | Set when course is first opened |
| completed_at | timestamp | Set when quiz is passed |
| quiz_score | int | Percentage score |
| quiz_passed | boolean | True if score meets pass threshold |

### `course_feedback`
| Field | Type | Notes |
|---|---|---|
| id | uuid | Primary key |
| bdr_id | uuid | FK → bdrs |
| course_id | int | 1–7 |
| rating | int | 1–5 Likert scale |
| comment | text | Optional free-text |
| created_at | timestamp | |

### `nudge_log`
| Field | Type | Notes |
|---|---|---|
| id | uuid | Primary key |
| bdr_id | uuid | FK → bdrs |
| nudge_type | text | `bdr_reminder` or `admin_alert` |
| sent_at | timestamp | |

### `sql_cache`
| Field | Type | Notes |
|---|---|---|
| id | uuid | Primary key |
| bdr_id | uuid | FK → bdrs |
| month_1_count | int | SQLs created in days 1–30 from onboarding start |
| month_2_count | int | SQLs created in days 31–60 from onboarding start |
| last_synced_at | timestamp | |

---

## Features

### 0. BDR Enrollment

- Admin-only screen (protected route).
- When a new BDR joins, the admin enrolls them before granting access to the academy.
- The enrollment screen fetches all contacts from HubSpot via the Contacts API and displays them in a searchable dropdown, showing each contact's full name and email.
- Admin selects the correct contact and clicks "Enroll."
- On enrollment, a `bdrs` record is created with:
  - `hubspot_contact_id` — HubSpot contact ID
  - `name` — contact's full name from HubSpot
  - `email` — contact's email from HubSpot
  - `onboarding_start_date` — today's date (the enrollment date)
- Because name and email are pulled directly from HubSpot at enrollment, the `endorser` matching for SQL tracking is guaranteed to be accurate.
- After enrollment, the BDR receives an email with access instructions (link to the academy).

### 1. Sequential Course Locking

- Courses are ordered 1–7.
- Course 1 is always unlocked.
- Course N unlocks only when course N-1 has `completed_at` set and `quiz_passed = true`.
- Locked courses display in a greyed-out state with the message: "Complete [Course N-1 name] to unlock."
- BDRs cannot access the deck, recording, or quiz of a locked course.

### 2. Course Completion & Quiz Tracking

- When a BDR opens a course for the first time, `started_at` is recorded in `course_progress`.
- When a BDR submits and passes the quiz, `completed_at`, `quiz_score`, and `quiz_passed` are recorded.
- Quiz pass threshold: **80%**.

### 3. Automated Nudges

**BDR nudge (email):**
- Triggered when a BDR has not recorded any new `completed_at` in the past 1 day.
- Only fires if the BDR has not yet completed all 7 courses.
- Email is sent to the BDR's registered email address.
- Logged in `nudge_log` with type `bdr_reminder`.
- Duplicate prevention: do not send if a `bdr_reminder` was already sent for this BDR in the past 24 hours.

**Admin alert (email):**
- Triggered when a BDR has not recorded any new `completed_at` in the past 2 days.
- Email is sent to the admin email address configured in environment variables (`ADMIN_EMAIL`).
- Logged in `nudge_log` with type `admin_alert`.
- Duplicate prevention: do not send if an `admin_alert` was already sent for this BDR in the past 24 hours.

**Cron schedule:** Run daily (e.g., 9am UTC). Check all active BDRs (those with `onboarding_start_date` set and fewer than 7 courses completed).

### 4. HubSpot SQL Tracking

**Data source:** HubSpot Deals API  
**Matching logic:** Query all deals where the custom property `endorser` equals the BDR's `name` field (exact string match). Because the BDR's name is pulled directly from HubSpot at enrollment, this match is guaranteed to be accurate.  
**SQL definition:** Any deal created with the BDR's name in the `endorser` property. Deal stage is not a factor — deal creation is the qualifying event.  
**Bucketing:**
- Month 1 SQLs: deals where `createdate` falls within days 1–30 from `onboarding_start_date`
- Month 2 SQLs: deals where `createdate` falls within days 31–60 from `onboarding_start_date`

**Sync schedule:**
- Daily via Vercel Cron (runs after nudge check)
- Also triggered on admin dashboard load (refreshes if `last_synced_at` is older than 1 hour)

**Caching:** Results stored in `sql_cache` to avoid hitting HubSpot API on every page load.

**Target:** 2 SQLs in month 1, 2 SQLs in month 2. Dashboard shows count vs. target.

### 5. Admin Dashboard

Protected route (admin-only). Shows a table with one row per BDR.

**Per-BDR columns:**
- Name
- Onboarding start date
- Courses completed (e.g., 4/7)
- Quiz scores per course (expandable or tooltip)
- Month 1 SQL count vs. target (e.g., 1/2)
- Month 2 SQL count vs. target (e.g., 0/2)
- Last active date (most recent `started_at` or `completed_at`)
- Nudge history (date of last BDR nudge and admin alert)

**Course feedback section:**
- Separate view (tab or section) showing per-course average rating and all submitted comments.

### 6. Course Feedback

- Shown to the BDR after completing each course (after quiz is passed).
- Two inputs:
  1. Likert scale rating: 1–5 ("How useful was this session?")
  2. Optional free-text comment ("Any other feedback?")
- Feedback is optional — BDR can skip.
- Stored in `course_feedback`.
- Visible to admin in the dashboard under the course feedback section.

---

## Business Rules Summary

| Rule | Value |
|---|---|
| Courses | 7, ordered sequentially |
| Unlock condition | Previous course completed + quiz passed |
| Onboarding start | Enrollment date (set by admin) |
| Month 1 window | Days 1–30 from onboarding start |
| Month 2 window | Days 31–60 from onboarding start |
| SQL definition | HubSpot deal created with BDR name in `endorser` |
| SQL target | 2 in month 1, 2 in month 2 |
| BDR nudge trigger | 1 day of no course completion |
| Admin alert trigger | 2 days of no course completion |
| Nudge dedup window | 24 hours |
| HubSpot sync | Daily + on dashboard load (if cache > 1 hour old) |
| Quiz pass threshold | 80% |

---

## Out of Scope

- Per-module action tasks
- Resource library
- Onboarding timeline view
- Weekly activity self-reporting
- BDR-facing SQL progress view
- Cohort benchmarking
