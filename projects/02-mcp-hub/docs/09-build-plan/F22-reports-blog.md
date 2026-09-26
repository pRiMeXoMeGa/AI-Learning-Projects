# F22: Reports, Blog & Video

| Milestone | Priority | Depends on | Effort | Unblocks |
|---|---|---|---|---|
| M6 | Must | F20, F21 | 3 h | Job applications |

**Goal:** Turn the work into things a hiring manager reads in five minutes: a strong README, three short
reports, a published threat model, a post and a video.

## Diagram: what gets published where

```mermaid
flowchart LR
    R1["reports/tool-design.md"] --> README
    R2["reports/security.md"] --> README
    R3["reports/performance.md"] --> README
    TM["docs/05 threat model"] --> README
    README["Repo README:<br/>what · architecture diagram ·<br/>3 headline numbers · install · demo link"] --> POST["LinkedIn / blog post<br/>(one finding, one chart)"]
    README --> VID["3-minute video:<br/>install server → login via gateway →<br/>confirmation → rug-pull quarantine → audit verify"]
    README --> CV["résumé bullets<br/>(03-projects template)"]
```

## Diagram: README structure

```mermaid
flowchart TB
    A["1. One-line pitch + badges (PyPI, npm, Registry)"] --> B["2. 30-second demo GIF"]
    B --> C["3. Architecture diagram"]
    C --> D["4. Headline numbers:<br/>ASR off → on · overhead p95 · best tool design"]
    D --> E["5. Install india-mf-mcp (Claude Desktop, VS Code)"]
    E --> F["6. Run the hub locally (compose)"]
    F --> G["7. Security model summary + link to threat model"]
    G --> H["8. What I'd do differently / build vs. buy"]
```

## Tasks
- [ ] Final runs of all three reports on the release commit
- [ ] README as above; demo GIF
- [ ] Blog/LinkedIn post: pick **one** finding (e.g. security ASR or tool-design result) with one chart
- [ ] 3-minute video
- [ ] Update the résumé bullet template in `03-projects.md` with real numbers

## Acceptance criteria
- A reader can install the server in under 2 minutes from the README
- Every number in the README links to the report and commit that produced it

**Interview talking point:** *"Every claim in the README links to a reproducible report, so the numbers
can be checked, not just trusted."*
