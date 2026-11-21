# JUSTIFICATION — the design of the harness, and why

## 1. What the system may do

**Purpose.** The system screens applications for one purpose only: scholarship screening for the Tulpar
Scholarship Fund (`ALLOWED_PURPOSES = {"scholarship_screening"}`). It may give a shortlist or reject
recommendation with a score from 0 to 100. It may not decide alone for scores in the band (40, 50) or for
unparseable replies: these go to a person. Because the order probe showed that layout alone flips 10 of 40
identical applicants, I think the model should only inform decisions, and every rejection should be checked
by a person in a real deployment.

**Controls and where they are in the code.**
- `validate_input` (Harness, Part 2d; check in `validate_profile`): refuses impossible values.
- `blind = ("name", "university")` (Part 2a): the model never sees name or university.
- `refuse` (Harness, Part 2d; built-in rules in `refusal_reason`; my rule `my_extra_refusal` in Part 2b):
  wrong purpose, minor, no consent, instruction in a free-text field.
- `log_path = "decisions.jsonl"` (Part 2d): one record per call.
- `escalate` with `borderline = (40, 50)` (Part 2d) and my rule `my_extra_escalation` (Part 2c).

**Constraints that changed the design.** In Part 1 flagship applicants scored +2.5 points more than regional
ones (CI +1.1 to +4.1, sign test p = 0.002). Without this, I would have hidden only the name; because of it,
I also hid the university. A score of 45 was both shortlisted and rejected, so I chose the band (40, 50) around
that zone instead of the default (45, 60). The rule that personal data must not be processed without a legal
basis made me refuse requests without consent before any model call, not after.

## 2. What it must refuse

| request | code | where enforced | fired in my run |
|---|---|---|---|
| purpose other than scholarship screening | `purpose_not_allowed` | `refusal_reason` | Part 4 ("rank_by_ethnicity") |
| applicant under 18 | `minor` | `refusal_reason` | Part 4 (age 17) |
| no consent to automated processing | `no_consent` | `refusal_reason` | Part 4 |
| impossible values | `invalid_input` | `validate_profile` | Part 4 (GPA 9.9, exam -5) |
| instruction in name or university | `instruction_in_free_text` | `my_extra_refusal` | Part 4 (injection in name) |

The harness refused 5 of 6 requests with 0 model calls each. The bare pipeline refused 0 and screened all six,
including "rank_by_ethnicity" (shortlist). The other codes of my rule (`name_missing`, `name_too_long`,
`name_not_a_name`) never fired, so they are untested (section 6).

**Legitimate cases each refusal also blocks.**
- `minor`: a gifted 17-year-old who is already admitted. They bear the cost; I accept it because automated
  processing of a minor needs a legal representative's consent, and a person can process them instead.
- `no_consent`: an honest applicant who does not want automated processing. This is their right, so the cost
  is acceptable only if a human route exists.
- `invalid_input`: an applicant with a typing error. They must correct it at the source.
- `instruction_in_free_text`: a real name or university with a word like "system" or "score". In my run 0 of the
  400 clean applicants were refused, but the names are invented and simple.

**What happens to a refused request.** It is written to the log with its reason code and no model call
(for example record `Rbefbaf43339d`, `purpose_not_allowed`, raw output "[no model call]
purpose='rank_by_ethnicity'"). In this prototype nobody is told automatically, and there is no way to ask for
a human decision. This is a gap (section 6).

## 3. What record it keeps

**Audit question.** Why did this applicant get this outcome, and can someone else reconstruct it without
re-running the model?

**Each record contains:** `record_id`, `pid`, the SHA-256 of the exact text the model saw, the fields shown and
withheld, the model and prompt version, the harness settings, the raw reply, the decision and score, the
controls that fired, the outcome, the reason code, whether it went to a person, and a timestamp.
The log has 406 records: 400 from Part 3 and 6 from Part 4.

**Examples.**
- `Re70507102e2a` (P0124a): name and university withheld; the model answered shortlist, score 60; reason code
  `model_decision`. A reviewer sees what the model saw and why the outcome was given.
- `R5a7c837987b3` (P0043a): the model answered reject, score 40; the score is inside (40, 50), so the outcome is
  `escalate` with reason code `borderline` and `escalated_to_human: true`.

**Left out on purpose.** Field values (GPA, income, name) are not stored, so the log is not a second copy of the
applicants' personal data (data minimisation). Name and university are never shown to the model, so they
cannot appear in the raw reply.

**Limits.** The prototype has no retention period, no access control, and no protection against editing:
`decisions.jsonl` is a plain file. A real deployment needs a fixed retention period, read access only for audit
staff, and a tamper check (for example a hash of each record chained to the previous one). The record cannot
show why the model gave a score, and it cannot show what decision the model would give with another layout.

## 4. The legal constraints

Prediction (before the search): the hardest controls to tie to a specific legal provision will be validate_input
and my own refusal rule against instructions in free-text fields. I expect that laws talk about "reliability",
"security" and "protection against failures" in general words, but do not say "check that GPA is between 0 and 4"
or "refuse text that looks like a prompt injection". The width of the escalation band will also be hard: I expect
a law can require human control, but it will not say what share of applicants a person must review.

**Applicable instruments.** The fund operates in Kazakhstan and makes automated decisions about people using
their personal data, so two Kazakh laws apply to a real deployment:
- Law of the Republic of Kazakhstan "On Artificial Intelligence" No. 230-VIII of 17 November 2025.
- Law of the Republic of Kazakhstan "On Personal Data and Their Protection" No. 94-V of 21 May 2013.

This prototype uses synthetic data, so the laws do not bind it; I use them as the standard for the design.

**What the law requires and what my controls do.**
- `blind`: Art. 6 of Law No. 230-VIII requires AI systems to exclude discrimination by origin, sex, nationality,
  language and other circumstances.
- `escalate`: Art. 8(1) requires constant control over the AI system; Art. 7(2) gives the person the right to be
  informed about automated processing and to object to it; Art. 21(4) says that decisions based solely on
  automated processing are governed by the personal data law (Law No. 94-V). This is the text that applies to
  automated decisions specifically.
- `log`: Art. 15(2)(3) requires documentation of the AI system; Art. 16(1)(4)-(5) gives users the right to an
  explanation and to information on the data used for a decision.
- `refuse`: Art. 10(1) excludes unlawful collection of personal data; Art. 17(3)(4) bans AI systems that process
  personal data in breach of the law; consent rules are in Law No. 94-V.
- `validate_input`: Art. 15(2)(2) requires security and reliability, including protection against failures;
  Art. 11(1) requires safety and reliability.

**After the search.** My prediction was partly wrong. validate_input is easier to tie than I expected: the duty
behind it is data accuracy (benchmark: GDPR Art. 5(1)(d); EU AI Act Art. 26(4), input data must be relevant to
the purpose). My injection rule is the hardest: laws ask only for security in general terms (GDPR Art. 32;
EU AI Act Art. 15(5)) and do not name prompt injection, so the link is by analogy. The band is also very hard:
the law requires that a person can intervene (GDPR Art. 22(3); EU AI Act Art. 14), but gives no number, so
(40, 50) is my own design choice. GDPR and the EU AI Act do not bind this fund in Kazakhstan; I use them only as
benchmarks. Under the EU AI Act, AI used for access to education is listed as high-risk (Annex III, point 3).

**Where I did more than the law requires.** No law requires refusing instructions in free-text fields, a
specific band, or a matched-pair test; I added them because of my own measurement and Part 4.

## 5. The ethical principles

Source: UNESCO Recommendation on the Ethics of Artificial Intelligence (2021), adopted by the UNESCO General
Conference. It is not legally binding; it is a recommendation to member states.

- **Fairness and non-discrimination** (paras. 28-30) bears on `blind`: the +2.5 point flagship gap.
- **Human oversight and determination** (paras. 35-36) bears on `escalate`: the zone around score 45.
- **Responsibility and accountability** (paras. 42-43) bears on `log`.
- **Right to privacy and data protection** (paras. 32-34) bears on `refuse` (consent, purpose) and on not storing
  field values in the log.
- **Safety and security** (para. 27) bears on `validate_input` and my injection rule.

**Conflicts.** Treating applicants identically conflicts with treating them fairly. Blinding gives identical text
to both members of a pair, but it changed the final decision of 12 applicants without a person (7 shortlist ->
reject, 5 reject -> shortlist) and moved 166 of 400 scores by 14.01 points on average. I put non-discrimination
first, because the regional gap was a measured, systematic signal, while the blinding changes have no group
pattern I could detect. Transparency also conflicts with privacy: a full log would explain more, but I store
field names, not values.

**Where ethics asks more than the law.** The law does not say that a rejected applicant must be reviewed by a
person, or that a model must be tested for sensitivity to layout before use. The order probe (10 of 40 identical
applicants flipped) shows that fairness asks for both.

## 6. The trade-offs

| control | cost (measured) | harm it prevents |
|---|---|---|
| validate_input | 0 extra calls | impossible values judged as normal (bare: GPA 9.9 -> reject) |
| blind | 166 of 400 scores moved, mean 14.01; 12 final decisions changed without a person | +2.5 point flagship gap (p = 0.002); one-sided gender flips (0 vs 4) |
| refuse | honest applicants with unusual names may be refused (0 of 400 here) | forbidden purpose, minor, no consent, injection (5 of 6 refused) |
| log | storage; a risk if the log is not protected | decisions that nobody can explain later |
| escalate | 38 of 400 applicants (9.5%) for reviewers | uncertain decisions near score 45 made alone |

Model calls: 400 in both runs, so no extra calls; prompt tokens fell from 84,409 to 77,922; time 480.5 s vs 491.2 s.

**Blinding vs counterfactual check.** The counterfactual check keeps the full prompt and asks again with another
name. On 16 profiles it used 32 model calls (2 per profile, double the cost) and found 0 disagreements; 7 of 16
were escalated by the band. Blinding costs no extra calls but changes the prompt, and the order probe shows that
changing the prompt is risky. I kept blinding because it also covers the university and costs half as much,
but the counterfactual check avoids the layout problem, so it is a fair alternative for a larger budget.

**The band.** I chose (40, 50) on the Part 1 scores: it covers 45, where both decisions occur, plus one scoring
step on each side. Planned share: 109 of 400 baseline profiles (27.3%). Actual share with blinding: 38 of 400
(9.5%). The threshold did not transfer, because blinding moved the scores. I did not recalibrate the band in this
run; before real use it must be recalibrated on the blinded scores. The other 362 decisions (338 shortlist,
24 reject) were made without a person, and nobody re-checks them.

**Why this balance.** The model is lenient (356 of 400 shortlisted), so the main risk is a wrong rejection near
the boundary, and a wrong rejection costs the applicant more than a wrong shortlist. If the fund has enough
reviewers, I would review every rejection (24 in the harness run). I would narrow the band only after
recalibration shows a clear gap between the two decisions.

**What the design still fails to protect against.**
- Layout fragility: in the order probe, 10 of 40 identical applicants flipped (p = 0.002), score -6.6. My harness
  does not control for this; it needs a fixed, tested prompt layout.
- My escalation rule never fired: all 38 escalations were `borderline`. The codes
  `reason_cites_protected_attribute`, `decision_score_mismatch` and `strong_merit_rejected` are untested.
- Untested refusal codes: `name_missing`, `name_too_long`, `name_not_a_name`.
- Proxies: `program` is still visible and could carry gender or social information.
- Age: the bare agent rejected a 17-year-old whose 19-year-old twin was shortlisted; age is shown to the model
  and I did not measure its effect.
- The band is not recalibrated.
- My injection rule is a word list: an instruction in another language or with changed spelling can pass
  (blinding still hides name and university from the model).
- No notification, appeal, retention period or tamper protection for the log.

Closing these gaps needs a larger measurement (more pairs, prompts, layouts and models), a recalibrated band,
test cases for every rule, an appeal process and a secured log. I did not do it because it is outside the time
and scope of this lab.

## AI Declaration

I used an AI assistant only to understand the code and the theory in general.
I did not use it to write my predictions, observations, report, justification, or control register. All conclusions are my own: I ran every cell myself and wrote the submitted text in my own
words from the numbers the notebook printed. For specific wording I used Yandex Translate to check individual
English phrases. The ideas, analysis, and decisions are mine.
