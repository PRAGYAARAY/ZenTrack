# ZenTrack
A burnout predictor and a smart timetable webapp.
# ZenTrack — Full System Architecture

> *"Know your limits before they know you."*

---

## 1. Project Overview

**ZenTrack** is a burnout prediction and smart timetable web app. Users answer a conversational quiz, write in a free-text stress diary, and receive:
- A **Burnout Risk Score** powered by ML + NLP
- An **auto-generated weekly timetable** built around their available time slots
- A **gamification layer** (XP, streaks, badges, levels)

**Target Users:** Students AND professionals (profile selected at signup)

---

## 2. Tech Stack

| Layer | Technology | Why |
|---|---|---|
| Frontend (UI Shell) | HTML + CSS + Jinja2 | Flask-rendered pages, fast SSR |
| Frontend (Interactive) | React (via CDN in Jinja templates) | Quiz flow, dashboard, timetable editor |
| Styling | Custom CSS + CSS Variables | One base theme now, user-switchable later |
| Backend | Python Flask | REST API + page rendering |
| Templating | Jinja2 | Server-side HTML for static pages |
| ML Model | Scikit-learn (Gradient Boosting or Random Forest) | Burnout risk score from structured inputs |
| NLP Model | HuggingFace `transformers` — `distilbert-base-uncased-finetuned-sst-2` | Sentiment analysis on diary text |
| Database | SQLite (dev) → PostgreSQL (prod) via SQLAlchemy | Users, sessions, scores, diary entries |
| Auth | Flask-Login + bcrypt | Simple session auth |
| Task Queue (optional v2) | Celery + Redis | Async NLP if model is slow |

---

## 3. Folder Structure

```
zentrack/
├── app/
│   ├── __init__.py              # Flask app factory
│   ├── models.py                # SQLAlchemy DB models
│   ├── routes/
│   │   ├── auth.py              # /login, /signup, /logout
│   │   ├── quiz.py              # /quiz — conversational input
│   │   ├── dashboard.py         # /dashboard — results + timetable
│   │   ├── diary.py             # /diary — daily stress diary
│   │   └── api.py               # /api/* — JSON endpoints for React
│   ├── ml/
│   │   ├── burnout_model.py     # Load + run sklearn model
│   │   ├── nlp_sentiment.py     # HuggingFace sentiment pipeline
│   │   ├── timetable_gen.py     # Timetable generation logic
│   │   └── train/
│   │       ├── train_model.py   # Training script
│   │       ├── dataset.py       # Synthetic + real dataset builder
│   │       └── burnout_model.pkl # Saved trained model
│   ├── static/
│   │   ├── css/
│   │   │   ├── base.css         # CSS variables, typography, reset
│   │   │   ├── quiz.css
│   │   │   └── dashboard.css
│   │   ├── js/
│   │   │   └── main.js          # Vanilla JS utilities
│   │   └── assets/              # Icons, illustrations
│   └── templates/
│       ├── base.html            # Base Jinja layout
│       ├── landing.html         # Landing page
│       ├── auth/
│       │   ├── login.html
│       │   └── signup.html
│       ├── quiz.html            # React mount point
│       └── dashboard.html       # React mount point
├── config.py
├── requirements.txt
├── run.py
└── README.md
```

---

## 4. Database Schema

### `users`
| Column | Type | Notes |
|---|---|---|
| id | INTEGER PK | |
| email | VARCHAR | unique |
| password_hash | VARCHAR | bcrypt |
| username | VARCHAR | display name |
| profile_type | ENUM | `student` / `professional` |
| xp_points | INTEGER | gamification |
| level | INTEGER | derived from xp |
| streak_days | INTEGER | consecutive daily logins |
| created_at | DATETIME | |

### `assessments`
| Column | Type | Notes |
|---|---|---|
| id | INTEGER PK | |
| user_id | FK → users | |
| taken_at | DATETIME | |
| profile_type | ENUM | student/professional |
| burnout_score | FLOAT | 0.0–1.0 (ML output) |
| risk_label | ENUM | low / moderate / high / critical |
| quiz_data | JSON | all quiz answers stored |
| sentiment_score | FLOAT | NLP output from diary |
| sentiment_label | VARCHAR | positive/negative/neutral |

### `diary_entries`
| Column | Type | Notes |
|---|---|---|
| id | INTEGER PK | |
| user_id | FK → users | |
| entry_text | TEXT | free-form diary |
| sentiment_score | FLOAT | |
| sentiment_label | VARCHAR | |
| created_at | DATETIME | |

### `timetables`
| Column | Type | Notes |
|---|---|---|
| id | INTEGER PK | |
| user_id | FK → users | |
| assessment_id | FK → assessments | |
| week_start | DATE | |
| schedule_data | JSON | generated timetable |
| created_at | DATETIME | |

### `badges`
| Column | Type | Notes |
|---|---|---|
| id | INTEGER PK | |
| user_id | FK → users | |
| badge_type | VARCHAR | e.g. `first_entry`, `7_day_streak` |
| awarded_at | DATETIME | |

---

## 5. ML Model Design

### 5A — Burnout Risk Score (Structured ML)

**Algorithm:** Random Forest Classifier (or Gradient Boosting)
- Outputs probability 0.0–1.0 mapped to: `low / moderate / high / critical`

**Student Features (Quiz Inputs):**
| Feature | Type | Range |
|---|---|---|
| avg_sleep_hours | float | 0–12 |
| study_hours_per_day | float | 0–16 |
| assignment_backlog | int | 0–20 |
| social_hours_per_week | float | 0–30 |
| exercise_days_per_week | int | 0–7 |
| self_rated_stress | int | 1–10 (quiz slider) |
| exam_proximity_days | int | 0–60 |
| skipped_meals_per_week | int | 0–21 |
| screen_time_hours | float | 0–18 |
| sentiment_score | float | NLP output (−1 to +1) |

**Professional Features (Quiz Inputs):**
| Feature | Type | Range |
|---|---|---|
| avg_sleep_hours | float | 0–12 |
| work_hours_per_day | float | 0–16 |
| meetings_per_day | int | 0–15 |
| deadline_count_this_week | int | 0–10 |
| vacation_days_last_month | int | 0–30 |
| self_rated_stress | int | 1–10 |
| wfh_days_per_week | int | 0–5 |
| skipped_meals_per_week | int | 0–21 |
| screen_time_hours | float | 0–18 |
| sentiment_score | float | NLP output |

**Training Data:** Synthetic dataset generated using domain-knowledge distributions + augmented with open-source datasets (e.g. Kaggle Burnout Dataset). ~5,000 samples minimum.

### 5B — NLP Sentiment (Deep Learning)

**Model:** `distilbert-base-uncased-finetuned-sst-2-english` (HuggingFace)
- Pre-trained transformer, no training needed
- Input: diary free text
- Output: `POSITIVE` / `NEGATIVE` + confidence score
- Mapped to: sentiment_score = +confidence (positive) or −confidence (negative)

**Pipeline:**
```
diary_text → tokenize → distilbert → sentiment_label + score → feed into ML model as feature
```

**Optional v2 Enhancement:** Fine-tune on a mental health/burnout corpus (e.g. Reddit `r/burnout` scraped dataset) using HuggingFace `Trainer`.

---

## 6. Timetable Generation Logic

**Inputs from user:**
- Available time slots per day (e.g. "Mon 9am–12pm, 3pm–6pm")
- Tasks/subjects to schedule
- Deadline dates per task
- Preferred focus block duration (25 min Pomodoro / 60 min deep work / custom)
- Break preferences

**Algorithm:**
1. Parse available slots → list of free time windows
2. Prioritize tasks by deadline proximity (urgency score)
3. Adjust intensity based on burnout score:
   - `high/critical` → more breaks, shorter blocks, leisure buffer
   - `low` → allow longer focus sessions
4. Fill slots greedily with highest-priority tasks
5. Inject mandatory breaks (Pomodoro ratio or custom)
6. Output: JSON schedule → React renders interactive calendar grid

---

## 7. Gamification System

| XP Event | Points |
|---|---|
| Complete daily diary entry | +20 XP |
| Complete full assessment quiz | +50 XP |
| Log into app (daily) | +10 XP |
| Stick to timetable (user confirms) | +30 XP |
| 7-day streak | +100 XP bonus |
| Burnout score improves | +75 XP |

**Levels:** 0→100 XP = Seedling 🌱 | 100→300 = Grower 🌿 | 300→600 = Focused 🧘 | 600→1000 = Zenith ⚡ | 1000+ = Enlightened ✨

**Badges:**
- 🔥 `First Flame` — Complete first assessment
- 📓 `Inner Voice` — Write 5 diary entries
- 💤 `Sleep Lord` — Log 8h sleep 7 days in a row
- ⚡ `Streak Master` — 14-day streak
- 📉 `Burnout Slayer` — Drop from high → low risk
- 🏆 `Zenith` — Reach max level

---

## 8. User Flow

```
Landing Page
    ↓
Sign Up (choose: Student / Professional)
    ↓
Onboarding Quiz (React — conversational, one question at a time)
    ├── Profile questions (sleep, hours, stress slider, etc.)
    └── Stress Diary (free text → NLP sentiment)
    ↓
ML Model runs → Burnout Risk Score generated
    ↓
Dashboard (React)
    ├── Burnout Score Card (gauge + label + explanation)
    ├── Sentiment Analysis result
    ├── Timetable Generator (input available slots → auto-schedule)
    ├── XP Bar + Level + Streak counter
    └── Badges earned
    ↓
Daily Return Loop
    ├── Quick diary entry (+XP)
    ├── Check timetable
    └── Re-assess weekly
```

---

## 9. API Endpoints

| Method | Route | Description |
|---|---|---|
| GET | `/` | Landing page |
| GET/POST | `/signup` | Registration |
| GET/POST | `/login` | Auth |
| GET | `/logout` | Logout |
| GET | `/quiz` | React quiz page |
| GET | `/dashboard` | React dashboard |
| GET | `/diary` | Diary entry page |
| POST | `/api/submit-quiz` | Submit quiz → run ML → return score |
| POST | `/api/submit-diary` | Submit diary → run NLP → return sentiment |
| POST | `/api/generate-timetable` | Input slots → return schedule JSON |
| GET | `/api/user-stats` | XP, level, streak, badges |
| GET | `/api/history` | Past assessments |

---

## 10. Aesthetic Direction

**Theme:** "Midnight Focus" — dark base, deep teal + soft amber accents
- Background: `#0D0F14` (near black)
- Primary accent: `#00C9A7` (bioluminescent teal)
- Secondary accent: `#FFB347` (warm amber — for warnings/burnout indicators)
- Text: `#E8EDF5` (cool white)
- Font: `Syne` (display) + `DM Sans` (body)
- Visual motif: Subtle particle/neural-network background on landing, smooth card transitions, glowing score gauge

**Planned v2:** Theme switcher (Light Zen / Dark Focus / Forest / Sunset) controlled via CSS variable swap.

---

## 11. Development Phases

### Phase 1 — Foundation
- Flask app setup, DB models, auth (login/signup)
- Landing page + quiz shell (HTML/Jinja)
- React quiz component (conversational flow)

### Phase 2 — ML Core
- Dataset generation + model training
- HuggingFace NLP pipeline integration
- `/api/submit-quiz` endpoint working end-to-end

### Phase 3 — Dashboard + Timetable
- React dashboard with score visualization
- Timetable generator UI + backend logic
- Results page with explanations

### Phase 4 — Gamification
- XP / level / streak system
- Badge awarding logic
- Dashboard gamification widgets

### Phase 5 — Polish
- Animations, transitions, loading states
- Theme switcher foundation
- Mobile responsiveness
- Testing + deployment

---

*Architecture v1.0 — ZenTrack*
