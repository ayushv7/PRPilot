# PRPilot — Evidence-Driven Pull Request Intelligence

> **"Know where to look before you review."**

PRPilot is an MCP (Model Context Protocol) server built with the official [NitroStack](https://nitrostack.ai) framework. It streamlines pull-request review by retrieving objective GitHub facts, analyzing file risk heuristics, verifying actual CI test results, tracing explicit issue references, and producing a structured, evidence-linked **Review Dossier**.

---

## The Problem

Code reviewers routinely lose context across fragmented GitHub views:
- **Diff volume & fatigue:** Sifting through hundreds of lines across dozens of files without knowing where core architectural changes lie.
- **Hidden risk boundaries:** Sensitive changes to authentication, configuration, data validation, or database migrations mixed in with routine changes.
- **Unverified CI assumptions:** Assuming tests passed when CI was actually never triggered, pending, or failed.
- **Disconnected requirements:** Reviewing code in isolation from the original issue report or acceptance criteria.

PRPilot investigates public GitHub pull requests, extracts verifiable facts, maps out architectural change areas, flags investigation signals, and generates a prioritized reviewer route.

---

## Architecture

```mermaid
graph TD
    User["Developer / AI Client<br/>(NitroStudio, Cursor, Gemini)"]
    MCP["PRPilot MCP Server<br/>(@nitrostack/core on Node.js)"]
    
    subgraph "Exposed MCP Tools"
        T1["inspect_pull_request"]
        T2["build_change_map"]
        T3["inspect_verification_evidence"]
        T4["trace_related_context"]
        T5["assemble_review_dossier"]
    end

    subgraph "Engine & Services"
        GH["GitHubService<br/>(Public REST API v3, Rate-Limit Aware)"]
        Analyzer["HeuristicsService<br/>(Categorization & Risk Heuristics)"]
        Dossier["DossierService<br/>(Deterministic Synthesis & Reviewer Route)"]
    end

    subgraph "Interactive UI"
        Widget["Review Dossier Widget<br/>(NitroStack Next.js / React)"]
    end

    User <-->|MCP Protocol (STDIO / SSE)| MCP
    MCP --> T1 & T2 & T3 & T4 & T5
    T1 & T2 & T3 & T4 --> GH
    T2 --> Analyzer
    T5 --> Dossier
    T5 -.->|@Widget('review-dossier')| Widget
```

---

## Core MCP Tools

All tools accept a validated input schema: `{ owner: string, repo: string, pull_number: number }`.

1. **`inspect_pull_request`**
   - Retrieves pull request metadata: title, description, author, branch targets, commit SHA, timestamps, and overall file statistics.
   - Accurately reports non-existent PRs and rate limits.

2. **`build_change_map`**
   - Categorizes changed files into: `Application Logic`, `Tests`, `Configuration & Dependencies`, `Documentation`, and `Other`.
   - Runs transparent heuristics on security-sensitive paths (`SIG_AUTH_SECURITY`, `SIG_DB_MIGRATION`, `SIG_CONFIG_ENV`, `SIG_DEPENDENCY_CHANGE`, `SIG_INPUT_VALIDATION`, `SIG_UNTESTED_LOGIC`, `SIG_HIGH_CHURN`).
   - Enforces diff bounding (2,000 character patch excerpts per file, 100 files max) and records truncation honestly.

3. **`inspect_verification_evidence`**
   - Retrieves real commit check-runs and statuses for the PR head commit SHA.
   - Distinguishes passing, failing, pending, and **missing/absent** checks (`no_checks_found`), never assuming an empty check list means tests pass.

4. **`trace_related_context`**
   - Parses explicit issue and PR references (`#123`, `gh-45`, URLs) from descriptions and commit messages.
   - Fetches issue titles, states, and links to ground the review in historical context.

5. **`assemble_review_dossier`**
   - Unifies all collected evidence into a cohesive, structured Review Dossier.
   - Produces a prioritized 3–5 step reviewer route with direct file links and rationale.
   - Powers the interactive **Review Dossier Widget** via `@Widget('review-dossier')`.

---

## Review Dossier Widget

Built with `@nitrostack/widgets` using Next.js and React:
- **Design:** Editorial technical aesthetic using a restrained graphite/charcoal palette (`#0d1117`, `#161b22`, `#21262d`), subtle borders, and monospace metadata for hashes, branches, and line numbers.
- **Interactive Tabs:**
  - **Reviewer Route:** Concrete 3 to 5 step guided path leading the reviewer from high-risk boundaries to test coverage and CI results.
  - **Investigation Signals:** Prioritized security, migration, and untested logic warnings.
  - **CI & Checks:** Real check runs and commit statuses with direct links.
  - **Evidence Ledger:** Clear separation between verified facts, heuristic signals, and unverified hypotheses.

---

## Quick Start

### Prerequisites
- Node.js 20.x or higher
- npm 9+

### Installation
```bash
# Clone the repository
git clone https://github.com/ayushv7/PRPilot.git
cd PRPilot

# Install root & widget dependencies
npm run install:all
```

### Environment Configuration
Copy `.env.example` to `.env`:
```bash
cp .env.example .env
```
*(Optional: add `GITHUB_TOKEN` to increase GitHub API rate limits from 60 to 5,000 requests/hr).*

### Running Locally
```bash
# Start MCP server and Widget dev server (hot-reload)
npm run dev

# Or build and run production server
npm run build
npm start
```

### Running Automated Tests
```bash
npm test
```
The test suite includes 14 tests covering GitHub API error shapes (404, 403 rate limits), file classification, diff truncation, honest absent-CI reporting, prompt injection safety, and a live smoke test against GitHub's public API.

---

## Testing in NitroStudio

1. Launch [NitroStudio](https://nitrostack.ai/studio).
2. Open the project directory (`PRPilot`).
3. NitroStudio connects automatically over stdio, discovering all 5 MCP tools and registering the widget preview.
4. In the **AI Chat Interface**, paste the investigation prompt:
   ```text
   Investigate facebook/react pull request #31000. Build the Review Dossier, explain which changes need the most careful human review, check the actual CI evidence, trace explicit related issues, and give me a prioritized review route. Cite the evidence for every important finding and tell me what you could not verify.
   ```
5. Observe the autonomous tool invocations and inspect the rendered Review Dossier widget.

---

## Deployment to NitroCloud

PRPilot is packaged for deployment to [NitroCloud](https://nitrocloud.ai):

```bash
# Create an optimized project archive
npx nitrostack-cli pack
```
This produces `prpilot.zip` excluding build artifacts, dependencies, and secrets. Deploy by importing the repository or uploading the package archive directly on the NitroCloud dashboard.

---

## Security & Integrity Principles

- **Read-Only MVP:** PRPilot does not modify repositories, post automated comments, approve pull requests, or merge code. The human reviewer remains in control.
- **Untrusted Input Protection:** All PR content (titles, bodies, commit messages) is treated strictly as data strings and isolated from execution flows.
- **Honest Evidence:** Missing CI or unavailable data is explicitly flagged as unknown/missing. PRPilot does not hallucinate test results or approvals.
