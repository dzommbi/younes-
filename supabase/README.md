# DLMed Supabase integration

The site uses the existing `subject`, `lessons`, `questions`, and `questions_answers` tables. No duplicate tables are defined here.

## Project inspection

- Project URL read from Supabase **Settings → API → Project URL**: `https://alfziyssfhiabqgmzlsb.supabase.co`.
- The browser publishable key is stored in `../supabase-config.js`. It is designed to be public; no secret or service-role key belongs in the website.
- All four existing tables have RLS enabled. The existing policies allow anonymous SELECT on all four tables, including `questions_answers`; that table currently exposes `correct_options` and `explanation` before an answer is submitted.
- All four tables currently contain zero rows. There are no existing subjects, lessons, questions, or corrections in Supabase to display.
- Required foreign keys exist. `questions_answers.question_id` currently has two equivalent foreign-key constraints; this integration does not remove or alter them.
- The app reads only question text and five choices from `questions`. It never selects `questions_answers` or the legacy `questions.explanation` column; corrections are requested through `grade_qcm_question` after submission.
- The existing local bank contains 590 QCM. It has been copied to the non-published workspace archive and exported to `../../work/supabase-import` as 17 JSON files. Export preserves the original question, choices, answer key, and explanation; no data was sent to Supabase.

## Files

- `../supabase-config.js`: project URL and public publishable key.
- `../supabase-data.js`: Supabase JS client queries, question normalization, offline read cache, and grading RPC call.
- `../supabase-runtime.js`: dynamic module/lesson picker and QCM session UI backed by Supabase while retaining the existing visual structure and controls.
- `migrations/20261104_secure_qcm_grading.sql`: reviewable SQL to remove public answer-key reads, restrict browser reads to non-correction question columns, and add the grading RPC. It creates no table and has not been executed.
- `catalog.json`: existing DLMed subjects/lessons for the optional first catalog seed; it creates metadata only, no questions.
- `../scripts/seed-qcm-catalog.ps1`: inserts only missing subject/lesson catalog rows and verifies them.
- `../scripts/export-existing-qcm.mjs`: copies all 590 current local QCM banks into JSON files under the workspace `work/supabase-import`; it does not write to Supabase.
- `../scripts/publish-qcm.ps1`: inserts questions and answer rows from JSON, skips exact duplicates, repairs a missing correction row, and verifies each submitted record. It never updates or deletes existing QCM.

## QCM JSON format

Each file contains one lesson. Preserve the French question, options, and explanation exactly:

```json
{
  "subject": { "name": "Sémiologie" },
  "lesson": { "title": "Partie 1 · Sémiologie de l’ECG" },
  "questions": [
    {
      "question": "Texte original de la question",
      "options": {
        "mode": "single",
        "choices": { "A": "…", "B": "…", "C": "…", "D": "…", "E": "…" }
      },
      "correct_options": ["C"],
      "explanation": "Explication originale"
    }
  ]
}
```

For multiple correct options, set `mode` to `multiple` and include every correct letter, for example `["A", "D"]`. The explanation and correct letters are stored only in `questions_answers`.

## Safe local publishing

Use PowerShell 7. Set the secret in the current process only; do not put it in the site, JSON files, GitHub, or chat:

```powershell
$env:SUPABASE_URL = 'https://alfziyssfhiabqgmzlsb.supabase.co'
$secret = Read-Host 'Supabase secret key (sb_secret_...)' -AsSecureString
$ptr = [Runtime.InteropServices.Marshal]::SecureStringToBSTR($secret)
try { $env:SUPABASE_SECRET_KEY = [Runtime.InteropServices.Marshal]::PtrToStringBSTR($ptr) }
finally { [Runtime.InteropServices.Marshal]::ZeroFreeBSTR($ptr) }

./scripts/seed-qcm-catalog.ps1
./scripts/publish-qcm.ps1 -Path ..\work\supabase-import
Remove-Item Env:SUPABASE_SECRET_KEY
```

`seed-qcm-catalog.ps1` and the publisher perform writes only when you run them. The publisher requires an already seeded exact subject/lesson unless you explicitly pass `-CreateCatalogIfMissing`.

## Database action still pending

The migration SQL, catalog seed, and QCM import have not been run. The database is empty, so the current 590-question bank will not appear in the live app until it is imported. The migration revokes anonymous/authenticated access to correction rows and exposes only the question columns used before submission. Apply the migration first, seed the catalog, and then publish `..\work\supabase-import`. Review and approve those production database writes before running them.
