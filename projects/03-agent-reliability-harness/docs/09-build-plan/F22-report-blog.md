# F22: Report, Blog & Video

| Milestone | Priority | Depends on | Effort | Unblocks |
|---|---|---|---|---|
| M6 | Must | F18, F19, F20 (F21 if built) | 4 h | Job applications |

**Goal:** Turn the results into things a hiring manager reads in five minutes: a strong README, the
framework comparison report, the DX scorecard, a post and a video.

## Diagram: what gets published where

```mermaid
flowchart LR
    R1["reports/framework-comparison.md"] --> README
    R2["reports/e5-crash-resume.md"] --> README
    R3["reports/failure-taxonomy.md"] --> README
    DX["DX scorecard<br/>(from the diary)"] --> README
    README["Repo README:<br/>pitch · headline table · pass^k chart ·<br/>'which framework when' · install · score your agent"] --> POST["LinkedIn / blog post<br/>(one finding, one chart)"]
    README --> VID["3-minute video:<br/>scenario starts → agent investigates →<br/>approval edit → resolved → report →<br/>same scenario in 4 frameworks side by side"]
    README --> CV["résumé bullets<br/>(03-projects template)"]
```

## Diagram: README structure

```mermaid
flowchart TB
    A["1. One-line pitch + badges (PyPI)"] --> B["2. Demo GIF (approval inbox)"]
    B --> C["3. Headline table (test split, CIs)"]
    C --> D["4. pass^k curves + failure-mode mix"]
    D --> E["5. Which framework I'd pick, when (DX + numbers)"]
    E --> F["6. Score your own agent (uvx opssim serve)"]
    F --> G["7. Method: scenarios, graders, stats, threats to validity"]
```

## Tasks
- [ ] Final report on the release commit; results hashes in the header
- [ ] DX scorecard table from the diary ([04 §4.10](../04-evaluation-design.md#410-developer-experience-dx-scorecard))
- [ ] "Which framework when" section: one paragraph per implementation, citing numbers
- [ ] README + GIF; blog/LinkedIn post with **one** finding and one chart
- [ ] 3-minute video (local recording if F21 is deferred)
- [ ] Update the résumé bullet template in `03-projects.md` with real numbers
- [ ] Publish the test scenarios with the report (hashes already committed, [ADR-016](../07-decisions.md))

## Acceptance criteria
- Every number in the README links to its report, results file and commit
- A reader can score their own agent from the README in under 5 minutes

**Interview talking point:** *"The report says where the comparison is weak: one engineer, a simulated
environment, two model families, and differences below about 8 points that it can't detect. Stating the
limits is why people trust the numbers."*
