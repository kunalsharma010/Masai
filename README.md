# CareLoop Clinics: No-Show Guard
 
Repo: [[FILL: public GitHub link]]
 
The question I was trying to answer: which appointments are likely to be missed, why do patients miss them, and what should CareLoop fix first?
 
## Quick summary
 
**Part 1, data.** The raw export had 1,020 bookings. After cleaning I was left with 990 rows I trust. I dropped 20 exact duplicates, 6 rows with dates that wouldn't parse and 4 with a blank status. I also fixed 5 impossible ages and 29 outliers. 218 of the 990 are no-shows, which is 22.0%.
 
**Part 2, SQL.** Lead time turned out to be the biggest factor. A booking made 31+ days ahead is missed 60.5% of the time. Booked 1–7 days ahead, that drops to 16.2%. Gomti Nagar Care Point in Lucknow has the worst clinic rate (33.3%), but it only had 90 appointments, so I wouldn't read too much into it.
 
**Part 3, model.** Logistic regression with a 0.35 threshold. On the test set it flags 49 of 198 bookings and catches 57.1% of the actual no-shows (recall 0.571, precision 0.571, ROC-AUC 0.789). Once the other features are accounted for, sending a reminder goes with roughly 69% lower odds of a no-show (odds × 0.31).
 
**Part 4, GenAI.** I ran the follow-up notes through Gemini using a CRAFT prompt, a JSON schema and a Pydantic check. All 9 non-blank notes came back with the reason I expected. The two notes tagged `forgot_or_no_reminder` were both from bookings that never got a reminder. Overall, 5 of the 10 notes came from no-reminder bookings.
 
## What I'd recommend
 
Short version: the bookings most at risk are the ones made far in advance, by patients who've missed before, who didn't get a reminder. Around one booking in five is missed (22.0%). For bookings made 31+ days out it's 60.5%, compared with 16.2% for 1–7 days.
 
1. **Make sure every booking gets a reminder.** This is the first thing I'd do. `reminder_sent` has the strongest negative coefficient in the model (odds × 0.31, so about 69% lower odds). Both "forgot / no reminder" notes also came from bookings that didn't get one (2 of 2).
2. **Have the front desk call the flagged bookings each evening.** At 0.35 the model flagged 49 of 198 test bookings and caught 57.1% of the real no-shows, and 57.1% of its flags were right. I'd start with long-lead bookings and patients with past no-shows, since `lead_days` (+0.61) and `prior_no_shows` (+0.65) have the largest positive coefficients.
3. **Look into Hadapsar Care Centre (27.3%) and Malviya Nagar Clinic (26.8%).** They're the two large clinics with the highest rates. Gomti Nagar is higher at 33.3%, but 90 appointments isn't enough to act on yet.
 
One caveat: this is all observational data, so the reminder result is an association. I'd run the reminder push as a trial and measure what it actually does.
 
## Why logistic regression
 
I compared four models using 5-fold cross-validated ROC-AUC on the training set only. The test set wasn't touched until the end.
 
| Model | CV ROC-AUC (train) |
|---|---|
| Logistic regression (StandardScaler + LogisticRegression pipeline) | **0.774** |
| Tuned random forest (`max_depth=3, n_estimators=200`) | 0.767 |
| Random forest | 0.747 |
| Decision tree (depth 3) | 0.678 |
 
I picked the threshold using out-of-fold probabilities from the training set. Moving from 0.5 to 0.35 raised F1 from 0.347 to 0.489 and recall from 0.249 to 0.467. The cost was 154 flags instead of 73. I leaned toward recall because a missed no-show means an empty doctor slot, while an extra phone call takes about a minute.
 
I scored the test set once at the end: ROC-AUC 0.789, precision 0.571, recall 0.571, with 49 of 198 bookings flagged (the scores are in `test_risk_scores.csv`). In practice the threshold should be set by how many calls the front desk can realistically make in an evening.
 
## Limitations
 
- The 10 notes only show that the pipeline works. Ten notes can't tell you why patients miss appointments across all the clinics.
- Reminders weren't randomly assigned, so the reminder effect is an association. It doesn't prove reminders cause fewer no-shows.
- The test results are probably a bit optimistic. I computed the cleaning stats (medians, quartiles, outlier fences) on all 990 rows before splitting. The same patient can show up in both train and test. And the split is random rather than past-to-future.
 
## Files
 
| File | Part | What's in it |
|---|---|---|
| `data/appointments.csv`, `data/clinics.csv`, `data/followup_notes.csv` | — | Original data, left unedited |
| `part1_data_foundations.ipynb` | 1 | Reading, validating, cleaning, enriching and exporting the data |
| `rejected_rows.json` | 1.1 | The 32 rows that failed type conversion |
| `clinics_enriched.csv` | 1.5 | Clinics with `temp_c` added from Open-Meteo |
| `appointments_clean.csv` | 1.6 | Clean table, 990 rows × 16 columns |
| `load_data.sql` | 1.6 | 998 INSERT statements (8 clinics + 990 appointments) |
| `part2_queries.sql` | 2 | Schema plus all the Part 2 queries (MySQL 8) |
| `part2_sql_analysis.ipynb` | 2 | The same SQL run in SQLite, with results and a pandas cross-check |
| `part3_ml_model.ipynb` | 3 | Baseline, logistic regression, trees, forest, and the model/threshold choice |
| `test_risk_scores.csv` | 3.5 | Risk score and flag for every test booking |
| `part4_genai_notes.ipynb` | 4 | Gemini extraction, Pydantic validation, prompt comparison, crosstab |
| `notes_analysed.csv` | 4.2 | The 10 analysed notes, with the raw replies kept |
 
## Notes on the Part 4 run
 
I used `gemini-3.5-flash` with temperature 0, JSON mode and a response schema. I had to remove `thinking_config` because this model rejected `thinking_level="minimal"`, even though the brief allows it. Gemini also threw two temporary 503 "high demand" errors, which the notebook retried automatically.
 
I counted every attempt, retries included, so the run used 14 API calls in total:
 
- Task 4.1: 1 call
- Task 4.2: 10 calls (9 notes + 1 retry)
- Task 4.3: 3 calls (2 V1 prompts + 1 retry). The V2 replies were reused from Task 4.2.
 
There's no API key anywhere in this repo. You type the Gemini key in when you run the notebook.
 