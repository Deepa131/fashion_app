# StyleSaathi — AI-Based Fashion Recommendation System

A full-stack web application that recommends outfits to Nepali women (18–26)
based on free-text fashion requests, powered end-to-end by the OpenAI API —
restricted so it only ever talks about fashion.

```
"I am attending a wedding and I want something elegant in red"
        ↓  (sent to OpenAI with a strict fashion-only system prompt)
{ occasion: "wedding", style: "elegant", color: "red",
  outfit: { top, bottom, footwear, accessory }, styling_tip }
```

If you ask it something unrelated to fashion ("what's the weather today?"),
it refuses and redirects you back to outfit recommendations — this is
enforced both in the prompt and validated again in the backend code.

---

## Tech Stack

| Layer          | Technology                              |
|----------------|------------------------------------------|
| Frontend       | React + Vite + Tailwind CSS + Axios       |
| Backend        | Python (FastAPI)                          |
| Database       | MongoDB (Atlas)                           |
| Auth           | JWT (python-jose) + bcrypt password hashing |
| AI / NLP       | OpenAI API (gpt-4o-mini by default) — single call does intent extraction AND outfit generation |

**An OpenAI API key is required** — there is no offline/static fallback by
design, per your requirement. See Setup below for where to put it.

---

## Project Structure

```
fashion-app/
├── backend/
│   ├── app/
│   │   ├── main.py                 # FastAPI app entrypoint
│   │   ├── config.py               # env var loading
│   │   ├── database.py             # MongoDB connection
│   │   ├── schemas/schemas.py      # request/response validation
│   │   ├── services/
│   │   │   ├── ai_engine.py        # OpenAI call + fashion-only guardrail + JSON validation
│   │   │   └── auth_dependency.py  # JWT auth guard
│   │   ├── routers/
│   │   │   ├── auth.py             # signup / login
│   │   │   ├── profile.py          # profile + preferences
│   │   │   ├── recommendations.py  # core AI recommendation endpoint
│   │   │   ├── outfits.py          # save / list / delete saved outfits
│   │   │   └── admin.py            # admin usage-stats dashboard
│   │   └── utils/                  # security + serialization helpers
│   ├── make_admin.py               # promotes an existing signed-up user to admin
│   ├── requirements.txt
│   └── .env.example
└── frontend/
    ├── src/
    │   ├── pages/                  # Home, Login, Signup, Recommend, SavedOutfits, Profile, Admin
    │   ├── components/              # Navbar, OutfitCard, ProtectedRoute
    │   ├── context/AuthContext.jsx
    │   └── api/client.js
    ├── package.json
    └── .env.example
```

There is no clothing catalog, no seed data, and no rule-based dictionary
NLP anymore — every recommendation is generated live by the OpenAI API.

---

## Setup — Backend

1. **Get an OpenAI API key**
   - Go to https://platform.openai.com/api-keys, create a key
   - Make sure your OpenAI account has billing/credits set up — without
     credits, calls will fail with a quota error (the app will show you
     a clear message if this happens, not a crash)

2. **Get your MongoDB Atlas connection string**
   - In Atlas: Database → Connect → "Drivers" → copy the `mongodb+srv://...` URI
   - Allow-list your IP under Network Access in Atlas (or `0.0.0.0/0` for dev)

3. **Configure environment variables**
   ```bash
   cd backend
   cp .env.example .env
   ```
   Open `.env` and fill in:
   - `MONGODB_URI` — your real Atlas connection string
   - `OPENAI_API_KEY` — your real OpenAI key
   - `JWT_SECRET_KEY` — any random string (the file shows how to generate one)

4. **Install dependencies**
   ```bash
   python -m venv venv
   source venv/bin/activate        # Windows: venv\Scripts\activate
   pip install -r requirements.txt
   ```

5. **Run the server**
   ```bash
   uvicorn app.main:app --reload --port 8000
   ```
   Visit `http://localhost:8000/docs` for interactive API docs.
   Visit `http://localhost:8000/health/db` to confirm MongoDB Atlas connects.

6. **(Optional) Make yourself an admin**
   Sign up through the app first, then:
   ```bash
   python make_admin.py your-email@example.com
   ```
   This just flips a flag on your account — it is not seed/catalog data.

---

## Setup — Frontend

```bash
cd frontend
cp .env.example .env       # default already points to localhost:8000
npm install
npm run dev
```
Visit `http://localhost:5173`.

For production build: `npm run build && npm run preview`

---

## How to Use It

1. Sign up
2. Go to "Get Outfit" and type something like:
   *"I need an outfit for college tomorrow. I like black clothes and casual style."*
3. The AI returns the extracted occasion/style/color plus a 4-piece outfit
   (top, bottom, footwear, accessory) and a styling tip
4. Try something off-topic, like *"what's the capital of Nepal"* — the AI
   will politely refuse and redirect you back to fashion, instead of
   answering. This is the guardrail in action.
5. Click "Save this outfit" — saves it to history AND nudges future
   recommendations toward your liked colors/styles (sent back to the AI
   as context on your next request)
6. "Saved" shows your outfit history. "Profile" shows your preferences and
   learned weights. Admin (if you ran `make_admin.py`) shows usage stats.

---

## How This Maps to Your Proposal

| Proposal requirement              | Where it's implemented                          |
|------------------------------------|--------------------------------------------------|
| NLP understands free-text input    | `ai_engine.py` — OpenAI extracts occasion/style/color |
| Styling logic generates an outfit  | `ai_engine.py` — same call generates the 4-piece outfit, constrained by the system prompt's rules |
| User preference modeling/learning  | `outfits.py` updates `preference_weights`; sent back into every future AI prompt as context |
| Personalized recommendations       | `/api/recommendations` endpoint                  |
| Web-based frontend                 | React app (`frontend/`)                          |
| Backend with database              | FastAPI + MongoDB Atlas                          |
| Save/manage outfits                | `/api/outfits` endpoints + Saved Outfits page    |
| Restricted to fashion domain only  | System prompt + schema validation in `ai_engine.py` (see "About the Guardrail" below) |

---

## About the Guardrail

The AI is restricted to fashion through two layers, not just a prompt
asking nicely:

1. **The system prompt** explicitly instructs the model to refuse anything
   not about clothing/outfits/styling, even if the user insists, claims an
   exception, or tries to redefine its role — and to respond with
   `is_fashion_related: false` in that case.
2. **Schema validation in code** (`_validate_shape` in `ai_engine.py`):
   the backend checks the JSON the model returns actually matches the
   expected outfit structure. If it doesn't, the backend treats it as
   invalid rather than trusting it blindly.

This two-layer design (prompt-level instruction + code-level validation)
is worth describing explicitly in your Methodology chapter — it shows you
understand that prompting alone isn't a reliable safety boundary.

---

## Troubleshooting

- **Recommendation request fails with "OpenAI rejected the API key"** →
  double check `OPENAI_API_KEY` in `backend/.env` — no quotes, no extra spaces.
- **"OpenAI rate limit or quota exceeded"** → check your OpenAI account has
  billing set up at platform.openai.com/account/billing
- **`/health/db` returns an error** → check your MongoDB Atlas IP allow-list
  and that `MONGODB_URI` has the correct username/password
- **Frontend can't reach backend** → check `VITE_API_BASE_URL` in
  `frontend/.env` and `ALLOWED_ORIGINS` in `backend/.env`
