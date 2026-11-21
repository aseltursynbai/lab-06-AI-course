# REPORT — Lab 06: The harness that says no

All numbers below are copied from the output of my executed notebook (Qwen2.5-3B-Instruct, greedy decoding, SEED = 2026, 40 pairs per attribute = 200 pairs = 400 profiles, one full run = 400 model calls). Nothing is estimated.

## 1. What I measured (bare agent, Part 1)

**Reliability.** 0 of 400 replies were unparseable. 356 of 400 profiles (89%) were shortlisted, 44 rejected.

**Flips** (the two members of a matched pair got different outcomes), read against the floor of the same kind of irrelevant change:

| attribute | flips | rate [95% CI] | its floor | floor flips |
|---|---|---|---|---|
| name_language | 2/40 | 5.0% [1.4, 16.5] | null_name | 0/40 (0.0% [0.0, 8.8]) |
| gender_coded_name | 4/40 | 10.0% [4.0, 23.1] | null_name | 0/40 |
| university | 1/40 | 2.5% [0.4, 12.9] | null_university | 1/40 (2.5% [0.4, 12.9]) |

For all three attributes the 95% intervals overlap their floors. **No attribute stood clear of its floor on decision flips.**

**Size of score differences** (mean |b − a|, 95% CI): name_language 3.12 [1.38, 5.25] vs null_name 2.00 [0.62, 3.62]; gender_coded_name 1.12 [0.25, 2.13]; university 2.50 [1.12, 4.12] vs null_university 2.12 [0.88, 3.62]. Again overlapping.

**Direction (signed):**
- *Flagship vs regional university:* flagship favoured in 1 pair, regional in 0 (McNemar p = 1.000), but the mean score gap is **+2.5 points [+1.1, +4.1]** (sign test p = 0.002). Mean score 63.2 (flagship) vs 60.8 (regional); shortlisted 92.5% vs 90.0%. This is the only signed effect whose interval excludes zero. It is small, and it did not turn into a measurable difference in decisions.
- *Male- vs female-coded name:* in all 4 discordant pairs the female-coded twin was shortlisted and the male-coded one was not (0 vs 4, McNemar p = 0.125); score gap −0.4 [−1.2, +0.6]; shortlisted 80.0% (male) vs 90.0% (female). Not significant — and with n = 40 it could not have been: the test needs at least 6 one-sided pairs (15%) to reach p < 0.05. I report it as **unresolved, not as absent**.
- *Name language:* Kazakh +0.7 [−1.8, +3.8], Russian +0.2 [−1.7, +2.1], English −1.0 [−4.2, +1.9] — all intervals contain zero.

**What 40 pairs can and cannot detect.** An exact McNemar test at 0.05 needs ≥ 6 pairs (15.0%) in one direction with none the other way, ≥ 8 (20.0%) with 1 the other way, ≥ 10 (25.0%) with 2 the other way. A real effect that moves fewer pairs than that is invisible to this test.

**Where the model is unstable regardless of who applies.** Rejected and shortlisted profiles share the 40–49 score range (33 rejected and 31 shortlisted profiles in that bin; shortlist minimum 45, reject maximum 45). Near the boundary the decision is close to arbitrary.

## 2. What the harness did (Parts 2–4)

Controls on: `validate_input`, `blind` (name, university), `refuse`, `log`, `escalate` (band 40–59 plus my own rule). `counterfactual_check` off.

- **Flips after blinding are 0/40 in every group — by construction.** Name and university are hidden, so both twins see identical text. This is not evidence that the model became fairer.
- **What changed for individual profiles** (bare → harness): shortlist 356 → 338, reject 44 → 24, escalate 0 → 38, refuse 0 → 0. 50 of 400 outcomes changed (reject→escalate 22, shortlist→escalate 16, shortlist→reject 7, reject→shortlist 5). **166 of 400 scores changed, mean |Δ| = 14.01 points.** That is far larger than the 2.5-point university gap I was trying to remove, and 12 profiles moved between shortlist and reject. Removing two lines changes the prompt's form as well as its content, so I cannot say the model "relied on" those fields; I can say the harness is not free.
- **Human workload:** 38 of 400 profiles (9.5%) were sent to a person. I had planned 110 of 400 (27.5%) from the baseline scores; the band was calibrated on unblinded scores and the blinded scores landed elsewhere. All 38 were `borderline`; my own escalation rule fired 0 times, and there were 0 unparseable replies.
- **Cost:** 400 model calls in both runs (1.00 per profile); prompt tokens 84,409 → 77,922; new tokens 11,122 → 11,088; seconds 470.6 → 467.2 (timing difference is noise). The cost of the harness is human reading time and changed decisions, not compute.
- **Refusal path (6 test requests):** the bare pipeline screened all six (it shortlisted the "rank by ethnicity" request and the prompt-injected name, and rejected the minor, the missing consent and the impossible values only because the model happened to score them low). The harness refused 5 of 6 with 0 model calls (`purpose_not_allowed`, `minor`, `no_consent`, `invalid_input`, `prompt_injection`) and screened the ordinary one. Refusal rules refused 0 of the 400 clean applicants — but those are clean synthetic profiles, so this says little about real false refusals.
- **Decision log:** `decisions.jsonl` has 406 records (400 measured + 6 test requests): 339 shortlist, 24 reject, 38 escalate, 5 refuse.

## 3. Predictions vs results

| prediction (written in the notebook) | result |
|---|---|
| 0: about 5–15 of 400 unparseable | **wrong**: 0 of 400 |
| 1: university tier diverges most; names near floor | **partly right**: university has the only signed score gap excluding zero (+2.5), but its flip rate equals its floor |
| 3: all flips 0 by construction; ~0 extra calls | **right** |
| 3: 25–35% escalated | **wrong**: 9.5% |
| 4: harness refuses 5 of 6 incl. the injection | **right**; 0 false refusals on 400 clean profiles |

(There is no written Prediction 2 in my notebook, so I do not report one.)

## 4. Who is affected, and how

- **Applicants from regional universities** lose on average about 2.5 points of suitability score against an otherwise identical flagship applicant. In this run that did not change the shortlist outcome more often than a harmless same-tier university swap, so the harm is a small systematic shift, not a proven rejection gap.
- **Applicants with male-coded names:** all 4 flips went against them. Not statistically established with 40 pairs; it needs a larger run before anyone can say "no effect".
- **Applicants near the threshold (scores 40–49):** the model's accept/reject there is close to arbitrary. These are the people the escalation band exists for.
- **Applicants whose decision is moved by the harness itself:** 50 of 400 had a different outcome after blinding, 12 of them between shortlist and reject. Some of these are people the old pipeline treated differently for prompt-formatting reasons.
- **Minors and applicants who did not consent:** the bare pipeline silently rejected them; the harness refuses to process them. A refusal is not a rejection: they still need a route (consent from a legal representative, or a human channel), otherwise the refusal just moves the exclusion.
- **Honest applicants wrongly refused:** 0 of 400 here, but my injection rule is a keyword list; real free-text could trip it, and a wrong refusal means the person is never evaluated.

## 5. Where my responsibility as the developer lies

1. **I own the claim I make.** I may say: "name and university can no longer influence the score, because the model does not see them." I may not say: "the system is fair." The remaining fields can still act as proxies, and I did not test that.
2. **I own the side effects of my own control.** Blinding moved 166 scores by 14 points on average. Before deployment I must re-calibrate the escalation band on the blinded scores, and re-measure with more pairs (the male/female result needs ≥ 6 one-sided pairs to be detectable at all).
3. **Reviewers are part of the system.** Escalation decides who gets a human reader, not what is fair. Someone must be assigned, trained and given enough time for ~10% of applications, and their decisions logged.
4. **Applicants' rights are a gap I have not built.** Kazakhstan's personal-data law (Art. 19-1, added by Law No. 231-VIII of 17 Nov 2025) says that for automated processing that affects a person's rights I must explain the procedure and consequences and give the person a way to object, and handle the objection within three working days. My harness has no notification and no objection channel yet.
5. **The log is evidence, so it must be kept responsibly:** it holds field names, not values, but it still carries application ids and input hashes. It needs access control and a retention period.
6. **Scope:** all of this is a synthetic, single-model, 40-pairs-per-attribute measurement. It is a method and a prototype, not a certification.
