# JUSTIFICATION — Lab 06: What the system may do, must refuse, and records

The Tulpar Scholarship Fund is fictional; I treat it as operating in Kazakhstan, so the binding instruments are Kazakh. The EU AI Act is mentioned only as an outside comparison. Numbers are from my executed notebook.

## 1. What the system may do

The agent may **score and recommend** (shortlist / reject) one application at a time, and only when all of these hold:

- the purpose is `scholarship_screening` (nothing else, e.g. not "rank by ethnicity");
- the applicant is 18 or older and has consented to automated processing;
- all values are inside valid ranges (`validate_input`);
- the model sees merit and need fields only: name and university are withheld (`blind`);
- the score is outside the 40–59 band and no other escalation rule fires — otherwise a person decides.

It may **not** be the final word on borderline cases. In the run, 338 profiles got an automated shortlist, 24 an automated reject, 38 (9.5%) went to a person.

*Legal basis:* Law of the Republic of Kazakhstan No. 94-V "On Personal Data and Their Protection" of 21 May 2013, Art. 7 p. 1 (processing with the subject's consent), Art. 7 p. 8 (collection and processing limited to specific, predetermined, lawful purposes), Art. 14 (use only for the declared purposes of collection); and Art. 19-1 p. 1 (automated processing from which rights or legitimate interests arise, change or end is prohibited unless the subject consented).

## 2. What it must refuse (and what the measurement says about each)

| refusal | reason code | tested | basis |
|---|---|---|---|
| purpose other than scholarship screening | `purpose_not_allowed` | refused, 0 model calls (bare pipeline shortlisted it) | 94-V Art. 7 p. 8, Art. 14 |
| applicant under 18 | `minor` | refused, 0 calls (bare rejected it) | 94-V Art. 7 p. 1, Art. 8 p. 1: consent comes from the subject *or the legal representative*; the system has no way to collect that, so it refuses and the case must go to a human channel |
| no consent to automated processing | `no_consent` | refused, 0 calls (bare rejected it) | 94-V Art. 19-1 p. 1 |
| impossible values (GPA 9.9, exam −5) | `invalid_input` | refused, 0 calls (bare rejected it) | data-quality control; see section 5 on why this is the hardest to tie to a statute |
| instruction hidden in a field | `prompt_injection` | refused, 0 calls (bare shortlisted it) | integrity of the decision (AI Law Art. 8, constant control); not a data-protection rule |

A refusal is not a rejection: the applicant must be told and offered another route. A wrong refusal denies an honest applicant any evaluation; on the 400 clean measured profiles my rules refused 0, but those profiles are synthetic and clean, so the false-refusal rate on real data is unknown. The injection rule is a keyword list and is the weakest refusal; the stronger protection is that name and university are blinded and never reach the model.

## 3. What record it keeps

`decisions.jsonl`: one line per call (406 lines: 400 measured + 6 refusal tests; 339 shortlist, 24 reject, 38 escalate, 5 refuse). Each line has a record id, application id, SHA-256 of the exact text the model saw, which fields were shown and withheld, model and prompt version, harness settings, raw reply, decision, score, controls fired, outcome, reason code, whether a person was involved, and a timestamp. It stores field **names**, never values, so it is not a second copy of applicants' data. A reviewer can say why each decision was made without re-running the model, and can answer an applicant's objection from the record.

*Open items:* retention period (94-V Art. 12 p. 2: data is kept until the purpose is reached) and access control for the log are not defined in this prototype.

## 4. Trade-offs I accepted

1. **Blinding instead of (or before) a counterfactual check.** Blinding adds no model calls (400 vs 400). The counterfactual extension would double the calls and leave the name visible to the model. Cost accepted: 166 of 400 scores changed (mean |Δ| 14.01) and 50 of 400 outcomes changed, 12 of them between shortlist and reject. I accepted this to close the channel for name and university, but it is a real cost and I do not call it "bias removed".
2. **A person reads ~10% of applications.** 38 of 400 (9.5%), lower than the 27.5% I planned (110 of 400) because the band was set on unblinded scores. I kept band 40–59 because rejected and shortlisted profiles overlap at 40–49 (33 and 31), and I will recalibrate on blinded scores before any real use.
3. **Strict refusal over auto-repair.** I refuse impossible, unconsented, underage or off-purpose requests instead of guessing or correcting them. Price: those applicants need a manual route.
4. **Small measurement.** 40 pairs per attribute detect only large effects (≥ 6 one-sided pairs, 15%). The male/female result (0 vs 4, p = 0.125) is therefore unresolved, and I did not hide a control because the test found nothing.
5. **Not built yet:** applicant notification and an objection channel with a three-working-day response (94-V Art. 19-1 p. 2–3).

## 5. Which control was hardest to tie to a provision (prediction vs finding)

*My prediction in the notebook:* `blind` and `escalate` would be hardest, because the personal-data law does not mention bias or human review.

*What I found:* the prediction was **wrong**. `blind` ties to 94-V Art. 7 p. 9 (data whose content and volume are excessive for the purpose is not to be processed) and Art. 12 p. 1 (only data necessary and sufficient); `escalate` ties to Art. 19-1 (explain, allow objection, a person looks) and to the AI Law's principles of controllability and priority of human decision-making. The **hardest** is `validate_input`: no article requires range checks on a GPA. I tied it to the safety and reliability principle of the AI Law and to OECD robustness, and I mark that link as the weakest in the register. Also, the `blind` argument is only partial: university can be a legitimate eligibility criterion for some scholarships, so blinding is justified for the scoring step, not necessarily for the whole process.

## 6. Instruments used (please verify before submission)

- Law of the RK No. 94-V of 21 May 2013 "On Personal Data and Their Protection" — consolidated text read at https://zakon.uchet.kz/rus/docs/Z1300000094 (Art. 5, 7, 8, 12, 14, 19-1, 22, 25 read in full; Art. 19-1 was added by Law No. 231-VIII of 17 Nov 2025).
- Law of the RK No. 230-VIII of 17 Nov 2025 "On Artificial Intelligence", in force 60 days after publication (https://zan.adilet.kz/ru/ob-iskusstvennom-intellekte-17-11-2025-no-230-viii/). I checked the article titles and the seven principles (Art. 4: legality, fairness and equality, transparency and explainability, responsibility and controllability, priority of human well-being and free will in decisions, data protection and confidentiality, safety and security; Art. 6 fairness and equality, Art. 7 transparency and explainability, Art. 8 responsibility and controllability, Art. 9 priority of human well-being). **I did not read the full text; open it and confirm the article numbers (especially 10 and 11) before you submit.**
- UNESCO Recommendation on the Ethics of Artificial Intelligence (2021), Section III.2 "Principles": fairness and non-discrimination; right to privacy and data protection; human oversight and determination; transparency and explainability. **Paragraph numbers in the register are the first paragraph of each principle from memory — check them in the PDF** (https://unesdoc.unesco.org/ark:/48223/pf0000381137).
- OECD AI Principles (2019, updated 2024), principle 1.4 robustness, security and safety.
- For comparison only, not binding here: Regulation (EU) 2024/1689 (AI Act) Art. 10, 12 and 14.
