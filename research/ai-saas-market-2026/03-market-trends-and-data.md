# 03 — Market Data and Trends (2026)

## 1. Consumer AI spending is real and growing

| Metric | Figure | Source (as reported) |
|--------|--------|----------------------|
| Gen-AI app in-app purchase revenue, 2025 | ~$5B+ (nearly tripled year over year) | Sensor Tower |
| Gen-AI app in-app purchase revenue, H1 2026 | >$4B (+36% vs H2 2025) | Sensor Tower |
| Apps mentioning "AI", in-app revenue H1 2026 | On track for ~$12B | Sensor Tower |
| Non-game app spending | Overtook games for the first time; ~$85B, +21% year over year, mostly driven by AI | Sensor Tower State of Mobile 2026 |
| Gen-AI app downloads | Doubled to 3.8B (2025) | Sensor Tower |
| ChatGPT | Reached 1B monthly active users in May 2026; ~2.5–2.7x larger than #2 (Gemini) | Sensor Tower / a16z |

**Takeaway:** consumers pay for AI. But spending concentrates in a few
winners, and generic chat is owned by ChatGPT and Gemini. New entrants have to
be vertical (one job, one audience).

## 2. The core problem: AI apps convert well but churn faster

RevenueCat *State of Subscription Apps 2026* (as reported):

| Metric | AI apps | Non-AI apps |
|--------|---------|-------------|
| Revenue per user | **+41%** | baseline |
| Churn speed | **~36% faster** | baseline |
| Year-1 realized LTV (median) | **$30.16** (top performers $49+) | $21.37 |
| Retention, monthly plans | 6.1% | 9.5% |
| Retention, annual plans | 21.1% | 30.7% |
| Refund rate | 4.2% | 3.5% |

**Takeaway:** novelty sells, but it doesn't keep people. Products that build
**measurable progress, habit or stored personal data** (a skill score, workout
history, a creative portfolio) resist this churn. This is the main reason I
weighted "retention potential" heavily in the niche scoring.

## 3. Inference costs (key for a $1k–10k budget)

| Modality | 2026 cost (published API prices) | Effect on a $10–20/month subscription |
|----------|----------------------------------|----------------------------------------|
| Real-time voice (speech in and out) | gpt-realtime-mini ~$0.016/min. Gemini Flash Live ~$0.036/min. Full gpt-realtime ~$0.05/min base, 2–5x on long sessions | 150 min/month ≈ **$2.50–7.50** → workable margins |
| Video generation | Seedance Fast ~$0.09/s. Veo 3.1 Lite $0.05–0.08/s. Kling/Sora 2 ~$0.10–0.15/s and up | One 30s clip ≈ **$2–12** before retries → pass cost on through credits |
| Image generation and editing | Cents per image. Commoditized by Nano Banana and others | Cheap, but hard to differentiate |
| On-device vision (pose, objects) | ~$0 (runs on the phone) | Best margins |
| Text LLM | Cents per thousand interactions on small models | Cheap |

**Takeaway:** voice and on-device vision fit your budget best. Video works only
if users pay per output.

## 4. Foundation-lab risk (the "wrapper" problem)

- Labs keep absorbing **horizontal** features: image editing (Nano Banana), slides (Gamma faced "RIP AI slides" commentary), general chat and generic voice chat.
- Labs have retreated from **expensive consumer entertainment** (Sora app shut down).
- Labs rarely build **vertical curricula, domain scoring, community or network effects**, or brand-specific workflows. That's the startup's space.

**Rule used in scoring:** if ChatGPT or Gemini could ship it as a feature in one
quarter, it scores low on lab risk.

## 5. Investor theses (what VCs want)

- **Y Combinator 2026:** consumer is only ~2–5% of recent batches, while YC is explicitly asking for "AI consumer products for a billion people". There's less competition for consumer investor attention.
- **a16z Big Ideas 2026:** consumer AI is moving "from productivity to connection". Products that **make expensive services cheap and accessible** (tutors, coaches, matchmakers, therapists, travel agents) are expected to do well. Their 6th Top-100 list shows AI is becoming the core of mainstream apps (Canva, CapCut, Notion).
- **Health and wellness:** fitness and wellness raised $3.6B in H1 2026, on pace for about +33% year over year. Investors now ask "what do you have that AI alone can't provide?"
- **Edtech:** overall edtech VC was down ~26% year over year (H1 2026), but AI tutors with clear outcomes keep raising (Speak, Ello, Buddy.ai, Coco Coders).

## 6. Regulation map (consumer-relevant)

| Area | Risk level | Notes |
|------|------------|-------|
| Companion chatbots | **High** | California SB 243 (minors' safeguards, in force 2026). Lawsuits involving teen harm |
| AI therapy | **High** | Banned or restricted in Illinois (fines up to $10k per violation), Nevada and Utah |
| Kids' products | Medium–High | COPPA, voice data of minors |
| Music and video generation | Medium | Copyright and likeness disputes. Disclosure rules for AI content |
| Adult skill coaching, fitness, productivity | **Low** | Standard privacy (GDPR/CCPA). Avoid medical claims |

## 7. Market size anchors used in the ideas

| Anchor | Figure | Used for |
|--------|--------|----------|
| People who speak English as a first or second language | ~1.53B; non-native speakers outnumber native ~3:1 | Idea 1 |
| Business English training market (2026) | ~$23B, ~8.8% CAGR to 2035 | Idea 1 |
| Digital English learning market (2026) | ~$16B, projected ~$31.6B by 2031 | Idea 1 |
| Independent musicians actively releasing music | ~9.5M; 125k+ tracks uploaded per day; artists spend ~18 hrs/week on marketing | Idea 2 |
| Suno paid subscribers | 2M | Idea 2 |
| US adults currently on GLP-1 drugs | ~12% (~16M active users) | Idea 3 |

These are market-research and secondary-source figures. Treat them as
order-of-magnitude anchors, not precise totals.
