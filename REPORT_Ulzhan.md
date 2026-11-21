# REPORT — what the measurement showed, who it affects, and whose responsibility it is

## 1. Measurement read-out

I measured the supplied agent (Qwen2.5-3B-Instruct, greedy decoding, prompt v1) on 40 matched pairs per
attribute (200 pairs, 400 profiles). All 400 replies were parsed (0 unparseable). The agent is very lenient:
356 of 400 profiles were shortlisted. Rejected profiles had scores 20-45 and shortlisted profiles had scores
45-85, so a score of 45 was both shortlisted and rejected.

| attribute | pairs | flips | flip rate [95% CI] | pairs with score diff | mean abs diff [95% CI] |
|---|---|---|---|---|---|
| name_language | 40 | 2 | 5.0% [1.4%, 16.5%] | 11/40 | 3.12 [1.38, 5.25] |
| gender_coded_name | 40 | 4 | 10.0% [4.0%, 23.1%] | 6/40 | 1.12 [0.25, 2.13] |
| university | 40 | 1 | 2.5% [0.4%, 12.9%] | 10/40 | 2.50 [1.12, 4.12] |
| null_name (floor) | 40 | 0 | 0.0% [0.0%, 8.8%] | 7/40 | 2.00 [0.62, 3.62] |
| null_university (floor) | 40 | 1 | 2.5% [0.4%, 12.9%] | 9/40 | 2.12 [0.88, 3.62] |

**Decisions.** No attribute stands clear of its own floor: for all three, the 95% interval of the flip rate
overlaps the interval of the floor, and the mean absolute score differences also overlap. Directions:
kazakh-language names 2 vs 0 (McNemar p = 0.500); russian 0 vs 1 and english 0 vs 1 (p = 1.000);
male-coded vs female-coded 0 vs 4 (p = 0.125), so all four gender flips favoured female-coded names
(shortlisted 90.0% vs 80.0%); flagship vs regional 1 vs 0 (p = 1.000).

**Scores.** One result is clear: flagship applicants got +2.5 points more than regional applicants
(CI +1.1 to +4.1, sign test p = 0.002). It is the only interval that does not include zero, and it stays
significant after dividing 0.05 by the five comparisons (0.01). The shift is small and usually does not
change the decision, because most scores are far from 45. All other score gaps include zero
(for example male vs female -0.4 [-1.2, +0.6]).

**What n = 40 can detect.** The exact McNemar test needs at least 6 of 40 pairs in one direction (15.0%) with
none the other way to reach p < 0.05; with 1 the other way it needs 8 (20.0%). The largest one-sided count
I saw was 4. So an effect that moves fewer than about 6 of 40 decisions cannot be detected by this run.
"No measured effect" is not "no effect".

**What the floors test and what they do not.** null_name changes which name is shown inside one language and
gender group; null_university changes the university inside one tier. Both keep the prompt wording, the field
order and the model fixed, so they do not measure sensitivity to the form of the prompt. Two other measurements
show that this sensitivity is large:
- **Order probe:** 40 pairs of identical applicants, the second shown with the same lines in another order.
  10 of 40 pairs flipped (25.0% [14.2%, 40.2%]), all in one direction (10 vs 0, McNemar p = 0.002), and the
  score fell by -6.6 points [-9.9, -3.5]. Only the layout changed.
- **Part 3:** removing two lines (name, university) changed the score of 166 of 400 profiles, with a mean
  change of 14.01 points.

So the agent reacts more to the layout of the prompt (10 of 40 flips) than to any protected attribute I measured
(at most 4 of 40). A stronger estimate would need more pairs, several prompt wordings and field orders, and more
than one model.

**Counterfactual check (extension).** On 8 name_language pairs (16 profiles, 32 model calls), swapping the name
to another language group never changed the decision (0 counterfactual_disagreement); 7 of 16 profiles were
escalated by the band, 9 were shortlisted. With only 16 profiles this cannot show that name language has no effect.

**Generalisation.** My run supports claims only about this model, this prompt, and invented names and
universities written in Latin script. It does not show how the agent treats real applicants, names in Cyrillic,
universities it has never seen, or a different model. Because the order probe shows that layout alone changes
25.0% of decisions, a different prompt wording could change all my results.

## 2. Who is affected

**Wrongly rejected applicants.** Regional-university applicants score 2.5 points lower than identical flagship
applicants. Usually this does not matter, but for an applicant near score 45 it can decide the outcome. In the
gender pairs, the 4 flips all went against male-coded names (not significant, p = 0.125). In Part 4 the bare
agent rejected a 17-year-old whose identical 19-year-old profile was shortlisted, so the model reacts to age
(one case only). A wrongly rejected applicant loses money and maybe a year of study, gets no reason, and nobody
re-checks the decision.

**Applicants affected by the layout.** The largest harm I measured is not linked to a protected attribute:
in the order probe, 10 of 40 identical applicants lost their shortlist only because the lines of the profile were
in another order. In Part 3, 12 applicants got a different final decision without a person (7 shortlist ->
reject, 5 reject -> shortlist) only because two lines were removed. These people are harmed or helped by an
arbitrary design choice, and they cannot know it.

**Wrongly shortlisted applicants and those they displace.** With 356 of 400 shortlisted, places become limited
later. Every wrong shortlist takes reviewer time and can push out another applicant. The cost is not equal:
a wrong shortlist is checked again later, but a wrong rejection is not.

**Parties the brief does not mention.** Reviewers: with my band (40, 50), 38 of 400 applicants (9.5%) were sent to
a person; on the unblinded scores the same band would hold 109 of 400 (27.3%). The fund carries the legal and
reputational risk. People who never apply because they expect to be rejected. Students from universities or
with names the model has never seen.

**If an effect exists below what I could detect.** An effect that moves up to 5 of 40 decisions would be
invisible here. Regional-university students would bear it first, because the only clear signal points at them.

**Compounding.** Used every year, a small regional penalty repeats for every cohort. If the shortlist is used as
input for a later review, another fund, or a ranking, the first decision is copied into the next one and the
original reason is lost. If the form of the application changes from year to year, decisions can change for
the same kind of applicant without anyone noticing.

## 3. Professional responsibility

**What I must check before it runs.** The prompt layout must be fixed and tested, because layout alone flipped
10 of 40 identical applicants. The escalation band must be recalibrated on the configuration that is really
used: it held 27.3% of unblinded profiles but sent only 9.5% after blinding. The matched-pair measurement must be
repeated after any change of model or prompt.

**What I must tell the deployers.** The scores are fragile: layout changed 25.0% of decisions in the order probe,
and removing two lines moved scores by 14.01 points on average. The measurement used only 40 pairs per attribute
and synthetic data. Blinding removes name and university, but program can still be a proxy. My own escalation
rule never fired in this run, so it is untested. The model should inform decisions, not make them.

**What I must not claim.** That the system is fair. The 0 flips after the harness are 0 by construction, because
both members of every pair see the same text; they do not prove that the model became fair.

**What I did in this lab.** I measured the bare agent, built five controls (validate_input, blind, refuse, log,
escalate), measured again, tested all refusals (5 of 6 requests refused with 0 model calls, 0 of 400 clean
applicants refused), kept a log of every call (406 records), and ran two extensions (counterfactual check and
order probe). Still left: recalibrating the band, testing my escalation rule, a larger sample, real-style names,
other prompts, and an appeal process.

**What belongs to the institution.** Setting and limiting the purpose, choosing and training the reviewers,
telling applicants that automated processing is used and how to object, handling appeals, and deciding how long
the log is kept and who can read it.

**Where the two conflict.** If the fund wants to remove escalation or blinding because it costs reviewer time,
I can show what each control prevents: escalation sent 38 applicants to a person, and blinding removed the
channel for the +2.5 point regional gap. If the fund still removes a control, I must document this decision and
the measured risk, and not present the system as safe without it. If the fund wants to use the scores as final
decisions, I must refuse to call that safe, because of the order-probe result.
