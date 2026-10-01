# 05 — Three AI SaaS Ideas

All three fit your constraints: a 2–5 person technical team, a $1k–10k budget,
B2C or prosumer customers, a global English-first market and a VC-scale goal.
Names are working titles only.

| | Idea 1 — **"Fluent"** | Idea 2 — **"DropKit"** | Idea 3 — **"FormCoach"** |
|---|---|---|---|
| One-liner | Real-time voice coach that makes non-native English professionals interview, present and lead meetings confidently | Upload a track, get a full release promo kit: lyric video, Spotify Canvas, 10 Shorts/Reels clips, cover art | Your phone watches your lifts and coaches you out loud, like a personal trainer at 1/50th of the price |
| Modality | Voice + text | Video + image + audio | On-device vision + voice |
| Niche score | 3.95 | 3.50 | 3.60 |
| Inference cost/user/month | ~$2.50–7.50 | ~$5–15 per promo kit (paid by credits) | <$1 |
| Main risk | Speak or ChatGPT voice moving in | Musicians' low budgets | Crowded fitness market |
| **Verdict** | **Recommended** | High upside, higher risk | Safe margins, harder to stand out |

---

## Idea 1 — "Fluent": voice coach for non-native English professionals (RECOMMENDED)

### Problem
Hundreds of millions of professionals work in English as a second language.
Technical skill isn't what holds them back. **Spoken confidence in high-stakes
moments** is: job interviews, client calls, presentations, salary negotiations,
stand-ups. Human communication coaches cost $50–200 an hour. General
language apps (Duolingo, Speak) teach the language, not workplace performance.

### Target user (first wedge)
Non-native English **job seekers and early-career professionals in tech,
finance and consulting**, worldwide. They're motivated (a job is on the line),
have urgent deadlines, and pay out of their own pockets. They're also
concentrated in reachable online communities (LinkedIn, r/cscareerquestions,
Blind, international student groups).

### Product (MVP)
1. **Scenario library:** behavioural interview, technical interview walkthrough, "tell me about yourself", salary negotiation, presenting a project, a difficult stand-up update. Each scenario is role-specific (SWE, PM, analyst, consultant).
2. **Real-time voice role-play** with an AI interviewer or manager that interrupts, asks follow-ups and pushes back like a real person.
3. **Instant scorecard** after each session: clarity, structure (STAR), filler words, pace, pronunciation of key terms, confidence markers. Includes the transcript and a "say it better" rewrite.
4. **Progress dashboard:** score trend over time, streaks, before-and-after recordings. This is the anti-churn layer.
5. **Bring your own context:** paste a job description or upload a CV, and the mock interview adapts to that role.

Positioning: **practice and preparation only**. Never a live, in-interview
"copilot". Hidden real-time interview assistants carry ethical and platform-ban
risk.

### Why now
- Real-time voice APIs reached human-like latency at ~$0.02–0.05 per minute in 2026.
- Speak at ~$100M ARR proves people pay for voice tutoring. Wispr and ElevenLabs show voice products are growing fastest.
- AI-driven hiring has made the market more competitive, so preparation matters more.

### Competition and differentiation
| Competitor | What they do | Our angle |
|---|---|---|
| Speak, Duolingo, Praktika | General language learning | We sell **career outcomes**, not vocabulary. We're for people who already speak English but need to perform in it |
| Final Round AI and interview copilots | Interview prep, some live in-interview assistance | We focus on **spoken delivery and confidence** for non-native speakers, with an honest practice-only position |
| ChatGPT and Gemini voice | Generic conversation | Structured scenarios, rubric scoring, progress tracking, role-specific content |
| Human coaches (Preply, Cambly, career coaches) | $15–200 per hour | 24/7 availability at 1/10th to 1/50th of the price |

### Moat over time
Proprietary data on **where non-native speakers struggle** (by first language,
role and scenario) improves the scoring models. Content network effects come
from a scenario library with user-submitted real interview questions. Distribution
partnerships with universities' international offices and bootcamps follow.

### Pricing
- Free: one full mock interview with a scorecard. The shareable score card is the viral loop.
- **Pro $19.99/month or $119.99/year.** **Sprint pass $39 for 30 days** (fits "my interview is in 3 weeks").
- Regional (purchasing-power) pricing for India, LatAm and Southeast Asia once demand appears there.

### Unit economics (estimate)
- Median usage ~150 voice minutes a month × ~$0.016–0.05 per minute ≈ **$2.50–7.50 a month**.
- Heavy users (600+ minutes) would cost $10–30, so use fair-use caps and route the bulk of practice to mini models.
- After app-store or Stripe fees, the target gross margin is **~65–75%**.

### Go-to-market (budget ≤ $10k)
1. **Founder-led content:** TikTok, YouTube Shorts and LinkedIn videos showing a real before-and-after interview answer. Non-native "career English" content performs well on these platforms.
2. **Shareable "Interview Readiness Score"** card after the free mock.
3. **SEO pages:** "Top 50 [role] interview questions, practise them out loud".
4. **Communities:** international student associations, coding bootcamps, job-seeker Discords and Reddit.
5. **Small paid tests ($1–2k)** on TikTok and Meta to measure cost to acquire a paying user.

### MVP plan (6–8 weeks)
| Week | Deliverable |
|------|-------------|
| 1 | Landing page and waitlist. Interview 20+ target users. Pick 2 roles and 5 scenarios |
| 2–4 | Web app (mobile-friendly): real-time voice role-play, transcript, scorecard v1 |
| 5–6 | Progress dashboard, payments, shareable score card, analytics |
| 7–8 | Launch to waitlist. Iterate on scoring quality. Start the content engine |

Suggested stack: Next.js and a mobile PWA (native iOS later). One provider for
real-time voice, with a fallback provider kept behind a model abstraction layer.
Postgres. Stripe. PostHog for analytics.

### Milestones to raise a pre-seed or seed round
- ~1,000 paying users or **$15–25k MRR**.
- Paid retention clearly better than the AI-app benchmark (beating 6.1% one-year retention on monthly plans and 21% on annual plans, as reported by RevenueCat).
- More than 40% of new users arriving organically. Evidence that users' scores improve over time.

### Path to VC scale
Interview prep (wedge) → workplace communication (meetings, presentations) →
**B2B2C**: employers and universities buy seats for international staff and
students → more languages (professional Spanish, German, Japanese).
At 500k paying users × ~$120 a year, that's ~$60M ARR before B2B seats.

### Risks and mitigations
| Risk | Mitigation |
|------|------------|
| Speak or Duolingo add professional modes | Go deep on career outcomes and role-specific content; move fast in the wedge |
| ChatGPT voice is "good enough" | Scoring, structure and progress are what users pay for; free chat isn't a coach |
| Scoring feels inaccurate | Calibrate the rubric against human coaches early; show evidence (timestamps) |
| Seasonal churn (users leave after getting the job) | Sprint pass plus a "career growth" mode after hiring; referral credits |

**Stop if** fewer than 3% of free users convert to paid, or paid churn exceeds
the RevenueCat AI benchmark after 3 months.

---

## Idea 2 — "DropKit": AI promo studio for independent musicians

### Problem
About 9.5M independent musicians release music, and 125k+ tracks are uploaded
every day. Artists reportedly spend **~18 hours a week on marketing versus ~12
hours making music**, and 84% do their own design and social media. Every
release needs visuals: cover art, a Spotify Canvas loop, a lyric video and a
steady stream of short clips for TikTok and Reels. Without them, the song gets
no attention.

### Target user
DIY artists who already pay for distribution or AI music tools (Suno has 2M
paid subscribers), and small labels or managers handling several artists.

### Product (MVP)
1. Upload a track and lyrics, or paste a DistroKid or Suno link.
2. **Beat- and section-aware analysis:** the AI finds the hook, drops and mood.
3. **One-click promo kit:**
   - Cover art (3 options in a consistent artist style)
   - Spotify Canvas (8s loop)
   - **Lyric video** (template motion graphics rendered programmatically, so it's cheap)
   - **10 vertical clips** cut on the hook using Higgsfield-style **viral format presets** ("POV", "story behind the song", "visualizer", "before vs after")
4. **Release calendar:** a 14-day posting plan with captions and hashtags.
5. Artist "style memory," so every release looks on-brand.

### Why now
Higgsfield (~$700M ARR) proved that **presets encoding viral formats** beat raw
prompting. Video model costs fell to ~$0.05–0.15 per second. AI music
(Suno at $300M ARR) created millions of new "artists" who need visuals.

### Competition and differentiation
Higgsfield, Captions and OpusClip are general-purpose. Canva has templates but
no understanding of music. Rotor, Kaiber and Neural Frames do music visuals but
not the full release workflow. **DropKit's angle: the only tool built around a
music release**, with beat sync, Canvas, lyric video, posting plan and artist
style in one flow.

### Pricing and unit economics
- **Per-release kit $29**, or **Pro $24/month** (2 releases + unlimited clips from templates).
- Cost per kit is ~$5–15: programmatic templates do most of the work, and only 2–3 clips use generative video. Target gross margin ~55–70%.
- Credits for extra generative clips, so heavy users pay their own compute.

### Go-to-market
Suno, Udio and producer Discords and subreddits. Musician creators on TikTok
and YouTube. Partnerships with distributors and beat marketplaces. A free
watermarked Canvas as the viral loop.

### Path to VC scale
Individual artists → labels and managers (team plans) → **distributor
integration** (sold through DistroKid, TuneCore and similar) → expand to podcasters and
other audio creators.

### Risks and mitigations
| Risk | Mitigation |
|------|------------|
| **Low willingness to pay**: 82% of independent artists reportedly earn <$1k a year from music | Target the serious top 10–20% and small labels; per-release pricing matches when they spend |
| Suno, Higgsfield or distributors build this | Move fast on the music-specific workflow; seek distributor partnerships, not just competition |
| Copyright and likeness | Users own their audio; no lookalike imagery of real artists; label AI visuals clearly |
| Video cost overruns | Templates first, generative video second; credits cap exposure |

**Stop if** the per-kit conversion from free Canvas users stays under 2%.

---

## Idea 3 — "FormCoach": camera + voice AI strength coach

### Problem
Strength training is booming, but beginners don't know whether their form is
safe. Personal trainers cost $60–120 a session. A specific, growing audience
needs this most: **~16M US adults on GLP-1 drugs**. Medical guidance commonly
pairs GLP-1 therapy with **resistance training and protein** to limit lean-muscle
loss. Most of the 568+ GLP-1 apps launched in 2026 are **trackers**, not coaches.

### Target user (first wedge)
Strength-training beginners, starting with people on GLP-1 drugs who want to
keep muscle and train at home or in the gym.

### Product (MVP)
1. **On-device pose estimation** (MediaPipe or Apple Vision) counts reps and checks joint angles: knee valgus, back rounding, depth.
2. **Voice coaching cues in real time** ("chest up", "two more, slow on the way down"), using pre-generated text-to-speech for low latency and near-zero cost.
3. **Adaptive weekly programme** from an LLM, based on goals, equipment, soreness and progress.
4. **GLP-1 mode:** protein targets, a muscle-preservation programme, symptom-aware deload days. Strictly wellness guidance, no medical claims.
5. **Progress:** strength curve, form score trend, streaks.

### Why now
On-device vision models are accurate and free to run. Voice output is cheap.
Cal AI showed one magical camera interaction can reach $50M ARR with 7 people.
GLP-1 adoption is creating a new audience that needs muscle preservation.

### Competition and differentiation
Fitbod and Ladder (programmes, no form feedback). Future (human coaches, ~$150+
a month). Many small 2026 form-check apps (FormFit, Riven, CoachAI), none
dominant. **Angle: the coach that sees you and talks to you, built
specifically for the GLP-1 and beginner audience**, with programmes plus form
correction plus voice in one product.

### Pricing and unit economics
- **$14.99/month or $89.99/year.** 7-day trial with a hard paywall (the Cal AI pattern).
- Cost <$1 per user per month (vision runs on the phone; LLM programming is weekly). Gross margin ~80%+ after app-store fees.

### Go-to-market
TikTok and Instagram fitness and GLP-1 creators (the Cal AI playbook). Form
check videos ("the AI caught my knee caving") are naturally viral. GLP-1
communities. Later, partnerships with telehealth GLP-1 providers.

### Path to VC scale
GLP-1 wedge → all beginner strength training → B2B2C through telehealth and
employer wellness → physical-therapy-adjacent rehab (with clinical partners).

### Risks and mitigations
| Risk | Mitigation |
|------|------------|
| Crowded fitness app market and high marketing cost | Stay in the GLP-1 wedge first; build a creator-led organic engine |
| Pose accuracy issues lead to bad cues | Limit to 15–20 well-supported lifts at launch; confidence thresholds; conservative cues |
| Injury or medical liability | Wellness positioning, disclaimers, no medical claims; escalate to "see a professional" |
| Acquisition before VC scale (like Cal AI) | Fine as an outcome; for VC scale, show B2B2C channels early |

**Stop if** trial-to-paid conversion stays under 25% with a hard paywall, or
week-4 workout retention stays under 30%.

---

## Final recommendation

**Build Idea 1 ("Fluent") first.**

- It has the highest niche score (3.95) and the best balance of growth evidence, cost and retention.
- Inference cost fits a $1k–10k budget, and users pay from day one.
- It suits a technical 2–5 person team: real-time voice and scoring are hard enough to deter copycats.
- There's a credible VC story: Speak shows the category can reach $1B; career outcomes plus B2B2C expansion lead to $100M+ ARR.

**If you prefer creative tools with bigger upside and more risk, choose Idea 2.** **If you want the lowest
costs and a proven paywall playbook, choose Idea 3.**

See [06-open-decisions.md](06-open-decisions.md) for the decisions still yours to make.
