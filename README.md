# SecureArc — Internal AppSec Platform
## Master Build Plan + AI Prompts
### Authored by: Senior Security Architect
### Model Recommendation: Claude Sonnet 4.5 (claude-sonnet-4-6 via API) for all generation phases

---

> **AI Era Reality Check**
> Traditional 8–12 week estimates assume manual coding.
> With Claude Code + iterative AI prompting, each phase compresses to **3–7 days** of focused iteration.
> Total estimated build: **6–10 weeks** with 1–2 developers running AI-assisted sprints.

---

## Product Vision

**SecureArc** is an internal AI-powered security architecture review platform that:
- Accepts design documents, architecture diagrams, and source code as input
- Performs automated gap analysis, threat modeling, and compliance mapping
- Detects design-to-code drift via GitHub PR integration
- Posts findings as structured PR comments and creates Jira tickets
- Maintains a full audit trail of all security decisions

---

## Which Claude Model to Use

| Task | Model | Why |
|---|---|---|
| All code generation (UI, backend, logic) | `claude-sonnet-4-6` | Best balance of speed + quality for iterative builds |
| Architecture reasoning, prompt design | `claude-opus-4-6` | Deeper reasoning for complex security logic |
| Lightweight PR diff analysis at scale | `claude-haiku-4-5` | Fast + cheap for high-volume webhook calls |

**For Claude Code CLI** (local terminal builds):
```bash
npm install -g @anthropic-ai/claude-code
claude  # launches interactive session
```
Use Claude Code for: refactoring entire modules, generating test suites, and wiring integrations.

---

## Tech Stack Decision

```
Frontend:    Next.js 14 (App Router) + Tailwind CSS + shadcn/ui
Backend:     FastAPI (Python) — AppSec team knows Python
Database:    PostgreSQL + pgvector (vector embeddings)
LLM layer:   Claude API (claude-sonnet-4-6) via LangChain
File store:  MinIO (local S3-compatible, zero cloud dependency for local dev)
Auth:        NextAuth.js (local dev: username/password, later: SSO/LDAP)
Queue:       Redis + RQ (background jobs for heavy analysis)
Infra (local): Docker Compose (entire stack in one command)
```

---

---

# PHASE 1 — Foundation UI + Document Upload + Basic Gap Analysis
## Target: Working local demo in 3–5 days

### What Phase 1 Delivers
- Dashboard homepage (project list)
- Create new project + upload design documents (PDF, DOCX, MD)
- AI-powered gap analysis on uploaded doc
- Gap report rendered in UI (severity-tagged findings)
- Basic user auth (login/logout)

### Phase 1 Validation Checklist
- [ ] Can upload a PDF architecture doc and get gap findings back
- [ ] Findings show severity (Critical / High / Medium / Low)
- [ ] Each finding has: description, affected component, recommendation
- [ ] UI is clean, navigable, no broken states
- [ ] Docker Compose brings up entire stack with one command
- [ ] No hardcoded secrets in codebase

---

### MASTER AI PROMPT — PHASE 1
> Paste this into Claude Code or a new Claude chat. This is your Phase 1 build prompt.

```
You are a senior full-stack engineer and security architect building an internal 
security architecture review platform called "SecureArc".

TECH STACK:
- Frontend: Next.js 14 (App Router), Tailwind CSS, shadcn/ui components
- Backend: FastAPI (Python 3.11)
- Database: PostgreSQL with pgvector extension
- LLM: Anthropic Claude API (model: claude-sonnet-4-6)
- File storage: MinIO (local S3-compatible)
- Auth: NextAuth.js with credentials provider
- Orchestration: Docker Compose

PHASE 1 GOAL: Build a working local development environment with:

1. DOCKER COMPOSE SETUP
   - postgres with pgvector
   - redis
   - minio
   - fastapi backend (port 8000)
   - nextjs frontend (port 3000)
   - All environment variables in a single .env file

2. DATABASE SCHEMA (PostgreSQL)
   Create tables:
   - users (id, email, password_hash, role, created_at)
   - projects (id, name, description, owner_id, created_at, status)
   - documents (id, project_id, filename, file_type, s3_key, uploaded_at, status)
   - findings (id, document_id, title, description, severity, category, 
               component, recommendation, status, created_at)
   - embeddings (id, document_id, chunk_text, embedding vector(1536), chunk_index)

3. FASTAPI BACKEND ENDPOINTS
   POST   /api/auth/login
   GET    /api/projects
   POST   /api/projects
   GET    /api/projects/{id}
   POST   /api/projects/{id}/documents   <- file upload to MinIO
   GET    /api/projects/{id}/documents
   POST   /api/documents/{id}/analyze    <- triggers gap analysis
   GET    /api/documents/{id}/findings
   GET    /api/health

4. GAP ANALYSIS ENGINE (core module)
   File: backend/services/gap_analyzer.py
   
   Function: analyze_document(document_id)
   - Download document from MinIO
   - Extract text (use pypdf for PDF, python-docx for DOCX, direct read for MD)
   - Chunk text into 1000-token overlaps
   - Generate embeddings via OpenAI-compatible endpoint OR store raw chunks
   - Send chunks to Claude API with this system prompt:
   
   SECURITY GAP ANALYSIS SYSTEM PROMPT:
   """
   You are a senior application security architect with 15 years of experience.
   You are reviewing a software architecture or design document to identify 
   security gaps BEFORE implementation begins.
   
   Analyze the provided document and identify security gaps across these domains:
   
   1. AUTHENTICATION & AUTHORIZATION
      - Missing MFA requirements
      - Weak session management
      - Privilege escalation risks
      - Missing RBAC/ABAC design
   
   2. DATA PROTECTION
      - Unencrypted sensitive data at rest
      - Missing TLS/encryption in transit
      - PII handling gaps
      - Key management not defined
   
   3. API SECURITY
      - Missing rate limiting
      - Unauthenticated endpoints
      - Insecure direct object references
      - Missing input validation design
   
   4. NETWORK & INFRASTRUCTURE
      - Missing network segmentation
      - Overly permissive firewall rules
      - Missing WAF/DDoS protection
      - Exposed management interfaces
   
   5. SECRETS & CONFIGURATION
      - Hardcoded credential risks
      - Missing secrets management (Vault, AWS Secrets Manager)
      - Environment variable handling
   
   6. LOGGING & MONITORING
      - Missing audit trail design
      - No alerting defined
      - Insufficient log retention
   
   7. SUPPLY CHAIN & DEPENDENCIES
      - No dependency scanning mentioned
      - Third-party integration risks
   
   8. COMPLIANCE GAPS
      - NIST CSF coverage gaps
      - PCI-DSS requirements not addressed
      - HIPAA safeguards missing (if health data involved)
   
   For each gap found, respond in this exact JSON format:
   {
     "findings": [
       {
         "title": "brief title of the gap",
         "severity": "CRITICAL|HIGH|MEDIUM|LOW",
         "category": "one of the 8 domains above",
         "affected_component": "which part of the system",
         "description": "clear explanation of the gap and why it matters",
         "recommendation": "specific actionable fix",
         "compliance_reference": "NIST CSF control or PCI requirement if applicable"
       }
     ],
     "summary": {
       "total_findings": n,
       "critical": n,
       "high": n,
       "medium": n,
       "low": n,
       "overall_risk": "CRITICAL|HIGH|MEDIUM|LOW",
       "top_priority": "one sentence on the single most important thing to fix"
     }
   }
   
   Be thorough. A finding missed at design time costs 100x more to fix post-deployment.
   """
   
   - Parse Claude response, store each finding in findings table
   - Update document status to "analyzed"

5. NEXT.JS FRONTEND PAGES
   /                        -> redirect to /dashboard
   /login                   -> login form (shadcn Card + Form)
   /dashboard               -> project list (shadcn DataTable)
   /projects/new            -> create project form
   /projects/[id]           -> project detail: document list + upload zone
   /projects/[id]/documents/[docId]  -> findings report page
   
   DASHBOARD DESIGN REQUIREMENTS:
   - Clean sidebar navigation (Projects, Settings)
   - Top bar with user avatar + logout
   - Project cards showing: name, document count, last analyzed, risk badge
   - Risk badges: red=CRITICAL, orange=HIGH, yellow=MEDIUM, green=LOW/PASS
   
   FINDINGS REPORT PAGE:
   - Summary bar at top: total findings, critical count, high count
   - Filter by severity and category
   - Each finding as an expandable card showing all fields
   - Export to PDF button (basic, Phase 1)
   - "Re-analyze" button

6. FILE: docker-compose.yml (complete, working)
7. FILE: .env.example (all required vars)
8. FILE: README.md with "Getting Started in 5 minutes"

CONSTRAINTS:
- All API keys come from environment variables only — never hardcoded
- Use async/await throughout FastAPI
- Frontend uses server components where possible
- Error states must be handled (upload fails, analysis fails, etc.)
- No placeholder data — everything must be wired end-to-end

Start by generating the complete docker-compose.yml and database schema, 
then the FastAPI backend, then the Next.js frontend. 
Generate complete, runnable files — not snippets.
```

---
---

# PHASE 2 — Threat Modeling + Compliance Mapper + GitHub PR Integration
## Target: 5–7 days after Phase 1 validated

### What Phase 2 Delivers
- STRIDE threat model auto-generation from uploaded design docs
- Threat model viewer in UI (interactive table + diagram)
- Compliance gap mapper (NIST CSF, PCI-DSS, HIPAA)
- GitHub App webhook listener
- PR security check runner (secrets, SAST via Semgrep, drift detection)
- PR comment poster with structured findings
- Jira ticket auto-creation for HIGH+ findings

### Phase 2 Validation Checklist
- [ ] Upload a design doc → STRIDE threat model generated with threats per component
- [ ] Compliance dashboard shows which controls are covered/missing
- [ ] Open a GitHub PR → webhook fires → analysis runs → comment posted within 90 seconds
- [ ] Critical finding on PR → merge blocked (required status check fails)
- [ ] Medium finding → Jira ticket created automatically
- [ ] Jira tickets link back to the finding in SecureArc UI

---

### MASTER AI PROMPT — PHASE 2
```
You are continuing to build SecureArc, an internal security architecture review platform.
Phase 1 (dashboard, upload, gap analysis) is complete and working.

PHASE 2 adds four major capabilities. Build each as a separate module:

═══════════════════════════════════════════
MODULE A: STRIDE THREAT MODELER
═══════════════════════════════════════════
File: backend/services/threat_modeler.py

Function: generate_threat_model(document_id)

Use this Claude system prompt for threat modeling:
"""
You are a senior application security architect specializing in threat modeling.
Using the STRIDE methodology, analyze this architecture document and generate 
a comprehensive threat model.

For each component or data flow mentioned, identify threats across:
- S: Spoofing (identity spoofing attacks)
- T: Tampering (data/code modification)
- R: Repudiation (denial of actions, audit gaps)
- I: Information Disclosure (data leaks, exposure)
- D: Denial of Service (availability attacks)
- E: Elevation of Privilege (unauthorized access escalation)

Also apply PASTA where relevant (Process for Attack Simulation and Threat Analysis):
- Map business objectives to technical risks
- Identify attack vectors an adversary would realistically use
- Prioritize by likelihood × impact

Respond in this JSON format:
{
  "components": [
    {
      "name": "component name from the document",
      "type": "web_app|api|database|service|user|external",
      "threats": [
        {
          "stride_category": "S|T|R|I|D|E",
          "threat_title": "concise threat name",
          "description": "what the threat is",
          "attack_vector": "how an attacker would exploit this",
          "likelihood": "HIGH|MEDIUM|LOW",
          "impact": "HIGH|MEDIUM|LOW",
          "risk_score": "CRITICAL|HIGH|MEDIUM|LOW",
          "mitigations": ["mitigation 1", "mitigation 2"],
          "mitre_attack_id": "T1xxx if applicable"
        }
      ]
    }
  ],
  "data_flows": [
    {
      "from": "component A",
      "to": "component B",
      "data_type": "what data flows",
      "threats": [...same structure as above...]
    }
  ],
  "trust_boundaries": [
    {
      "boundary_name": "e.g. Internet / DMZ",
      "crossing_components": ["list of components crossing this boundary"],
      "risks": ["key risks at this boundary"]
    }
  ],
  "executive_summary": "2-3 sentences on the highest priority threat areas"
}
"""

Frontend page: /projects/[id]/threat-model
- Trust boundary diagram (SVG-based, generated from threat model data)
- Component table with STRIDE coverage per component
- Threat list filterable by STRIDE category, risk score, component
- Each threat expandable with full details + mitigations
- MITRE ATT&CK link when ID is present

═══════════════════════════════════════════
MODULE B: COMPLIANCE MAPPER
═══════════════════════════════════════════
File: backend/services/compliance_mapper.py

Pre-load these framework controls into the database (controls table):
- NIST CSF 2.0: all 106 subcategories across Govern/Identify/Protect/Detect/Respond/Recover
- PCI-DSS v4.0: all 12 requirements broken into sub-requirements
- HIPAA: Security Rule safeguards (Administrative, Physical, Technical)

Function: map_findings_to_controls(project_id)
- For each finding in the project, use Claude to map it to relevant controls
- Score each framework: controls_addressed / total_controls * 100
- Store compliance_scores table (project_id, framework, score, gap_count, assessed_at)

Frontend page: /projects/[id]/compliance
- Three framework cards: NIST CSF / PCI-DSS / HIPAA
- Each shows overall % coverage with a circular progress indicator
- Drill-down: which controls are COVERED / PARTIAL / MISSING
- MISSING controls listed with which findings address them
- Export compliance report as PDF

═══════════════════════════════════════════
MODULE C: GITHUB PR INTEGRATION
═══════════════════════════════════════════

Step 1: GitHub App setup instructions (README section)
Step 2: Webhook receiver

File: backend/routers/github_webhook.py

POST /api/webhooks/github
- Verify GitHub webhook signature (HMAC SHA-256)
- On pull_request events (opened, synchronize, reopened):
  - Extract: PR number, repo, head SHA, diff URL, base branch
  - Queue background job: analyze_pr(pr_data)

File: backend/services/pr_analyzer.py

Function: analyze_pr(pr_data)
1. Fetch the PR diff via GitHub API
2. Run checks in parallel:

   CHECK 1: Secret scanning
   - Regex patterns: AWS keys, GitHub tokens, private keys, connection strings
   - High-entropy string detection (Shannon entropy > 4.5 on strings > 20 chars)
   
   CHECK 2: Dependency check  
   - Detect changes to: package.json, requirements.txt, pom.xml, go.mod, Gemfile
   - For each changed dependency file, call OSV API for CVE lookup
   - Flag any dependency with CVSS >= 7.0
   
   CHECK 3: SAST via Semgrep
   - Run semgrep with these rulesets: p/owasp-top-ten, p/secrets, p/python (if Python)
   - Parse semgrep JSON output
   - Map findings to affected files + lines in the diff
   
   CHECK 4: Drift detection (your key differentiator)
   - Retrieve the project's embedded design doc from vector DB
   - Extract changed files and new patterns from diff:
     * New API routes/endpoints added
     * New database queries
     * New external HTTP calls
     * Changes to auth/middleware
     * New file upload handlers
     * New environment variable usage
   - For each detected pattern, query vector DB: "is this consistent with the design?"
   - Use Claude to assess: "Does this PR introduce components/patterns not present 
     in the original architecture document? Flag any drift with explanation."
   
   CHECK 5: Auth logic detection
   - Detect changes to files matching: *auth*, *middleware*, *permission*, *rbac*, 
     *session*, *token*, *oauth*, *login*, *password*
   - Send changed auth-related code to Claude:
     "Review this auth code change. Does it weaken any security control? 
      Could it allow authentication bypass or privilege escalation?"

3. Aggregate all findings into a severity-ranked list
4. Post GitHub PR comment (see format below)
5. Set GitHub commit status:
   - Any CRITICAL finding → status: failure, description: "SecureArc: Critical security issue"
   - HIGH findings only → status: failure (configurable threshold in settings)
   - MEDIUM/LOW only → status: success with warning in comment
6. For HIGH+ findings → create Jira ticket via Jira REST API

GITHUB PR COMMENT FORMAT:
"""
## 🔐 SecureArc Security Review

{if critical} ❌ **MERGE BLOCKED** — {n} critical finding(s) require AppSec review
{if high only} ⚠️ **Security warnings** — {n} high severity finding(s) found
{if pass} ✅ **Security checks passed** — {n} low-severity notes

### Findings Summary
| Severity | Count | Check |
|----------|-------|-------|
| 🔴 Critical | n | Secret detected / Auth bypass |
| 🟠 High | n | CVE in dependency / Drift detected |
| 🟡 Medium | n | Missing input validation |
| 🔵 Low | n | Logging gap |

### Details
{for each finding, one collapsible section}
<details>
<summary>🔴 [CRITICAL] Hardcoded AWS Secret Key — src/config.py:42</summary>

**What was found:** AWS_SECRET_ACCESS_KEY hardcoded in source file  
**Why it matters:** Exposes cloud credentials if repo is accessed  
**Fix:** Move to environment variable or AWS Secrets Manager  
**Jira:** [SEC-{ticket_number}]({jira_link})
</details>

---
*Powered by SecureArc | [View full report]({securearc_link}) | [Project settings]({settings_link})*
"""

═══════════════════════════════════════════
MODULE D: JIRA INTEGRATION
═══════════════════════════════════════════
File: backend/services/jira_client.py

Settings stored in DB: jira_url, jira_project_key, jira_email, jira_api_token (encrypted)

Function: create_security_ticket(finding)
- Create Jira issue with:
  - Issue type: Bug or Security (configurable)
  - Priority: maps from CRITICAL→Highest, HIGH→High, MEDIUM→Medium
  - Summary: "[SecureArc] {finding.title}"
  - Description: full finding details in Jira markdown
  - Labels: ["security", "appsec", finding.category.lower()]
  - Custom field: SecureArc finding ID (for dedup)
- Return ticket key, store in finding record

Frontend: Settings page /settings/integrations
- GitHub App: connection status, repos connected, webhook URL
- Jira: URL, project key, auth token (masked), test connection button
- Notification preferences: which severity creates tickets

Generate complete, wired, runnable code for all four modules.
```

---
---

# PHASE 3 — Drift Detection Deep Dive + Runtime Mapping + CI/CD Gate
## Target: 5–7 days after Phase 2 validated

### What Phase 3 Delivers
- Deep design-to-code drift analysis (beyond PR diffs — full codebase scan)
- Runtime architecture mapper (AWS/GCP/Azure asset discovery)
- CI/CD pipeline gate (GitHub Actions workflow template)
- Drift timeline view: how architecture has evolved vs original design
- Risk scoring dashboard with trends over time

### Phase 3 Validation Checklist
- [ ] Full codebase scan detects components not in the design doc
- [ ] Cloud asset discovery pulls real deployed resources
- [ ] Comparison: what's in the design vs what's actually deployed
- [ ] GitHub Actions workflow template works copy-paste
- [ ] Risk score trending chart shows improvement over time
- [ ] Full audit log of all analysis runs, findings, and resolutions

---

### MASTER AI PROMPT — PHASE 3
```
You are completing Phase 3 of SecureArc, an internal security architecture review platform.
Phases 1 (dashboard + gap analysis) and 2 (threat modeling + PR integration) are complete.

PHASE 3 adds three final capabilities:

═══════════════════════════════════════════
MODULE E: DEEP DRIFT ANALYSIS (Full Codebase)
═══════════════════════════════════════════
File: backend/services/drift_analyzer.py

This is different from PR drift (which checks one PR).
This scans the ENTIRE current codebase against the design document.

Function: full_drift_analysis(project_id, repo_url, branch="main")

1. Clone or fetch the repo (use GitHub API to get file tree + content)
2. Build a "code architecture map" by analyzing:
   - All API route definitions (FastAPI, Express, Django URLs, Spring mappings)
   - All database model/schema files
   - All external service clients (HTTP calls, SDK usage)
   - All authentication/middleware files
   - All configuration/environment variable references
   - Infrastructure as code (Terraform, CloudFormation, k8s manifests)

3. Compare code architecture map against design document embeddings:
   Use Claude with this prompt:
   """
   You have two inputs:
   A) The original architecture design document (what was planned)
   B) The current codebase architecture map (what was actually built)
   
   Identify ALL drift between design intent and implementation:
   
   1. PRESENT IN DESIGN, MISSING IN CODE
      Components/controls planned but never implemented
   
   2. PRESENT IN CODE, MISSING IN DESIGN
      Components built without design approval (shadow architecture)
   
   3. IMPLEMENTED DIFFERENTLY THAN DESIGNED
      Components exist in both but work differently than specified
   
   4. SECURITY CONTROLS DESIGNED BUT WEAKENED IN CODE
      Security measures specified in design but implemented incompletely
   
   Return JSON with drift_items array, each with:
   - drift_type: MISSING_IMPLEMENTATION|SHADOW_COMPONENT|IMPLEMENTATION_MISMATCH|WEAKENED_CONTROL
   - severity: CRITICAL|HIGH|MEDIUM|LOW
   - title, description, design_reference, code_reference, recommendation
   """

4. Store drift findings separately from gap analysis findings
5. Track drift over time (each run creates a snapshot)

Frontend: /projects/[id]/drift
- Timeline view: drift count over time (line chart)
- Current drift items grouped by type
- "Design intent" vs "Actual implementation" side-by-side for each item
- Resolve drift: mark as "Accepted deviation" with justification or "Will fix"

═══════════════════════════════════════════
MODULE F: RUNTIME ARCHITECTURE MAPPER
═══════════════════════════════════════════
File: backend/services/cloud_mapper.py

Cloud providers to support (detect from project settings):
- AWS: use boto3 with read-only IAM role
- GCP: use google-cloud SDK with viewer role  
- Azure: use azure-sdk with Reader role

Discover:
- Compute: EC2/GCE/VMs, ECS/GKE/AKS containers, Lambda/Cloud Functions
- Network: VPCs, subnets, security groups, load balancers, API gateways
- Storage: S3/GCS/Blob buckets (check: public access? encryption?)
- Databases: RDS/Cloud SQL/Azure SQL, DynamoDB, Redis
- IAM: roles, policies (flag overly permissive: *, admin roles)
- DNS/CDN: Route53, CloudFront, Cloud CDN

Build a "runtime architecture snapshot":
- JSON map of all discovered resources with security-relevant properties
- Flag immediate security issues: public S3 buckets, unencrypted DBs, open security groups (0.0.0.0/0)

Compare runtime snapshot against design document:
- Resources in design but not deployed (incomplete implementation)
- Resources deployed but not in design (shadow infrastructure)
- Security properties different from design spec (e.g., design says encrypted, actual is not)

Frontend: /projects/[id]/runtime
- Visual topology map (D3.js or Cytoscape.js) of discovered cloud resources
- Color-coded: green=matches design, yellow=partial match, red=not in design or security issue
- Resource detail panel on click
- Security issue count per resource type

═══════════════════════════════════════════
MODULE G: CI/CD PIPELINE GATE + RISK DASHBOARD
═══════════════════════════════════════════

Generate a GitHub Actions workflow template:
File: .github/workflows/securearc-security-gate.yml

The workflow should:
1. Trigger on: pull_request (opened, synchronize) and push to main
2. Steps:
   - Checkout code
   - Run Semgrep (using returntocorp/semgrep-action)
   - Run gitleaks for secret scanning
   - Run osv-scanner for dependency CVEs
   - POST results to SecureArc API endpoint: /api/ci/results
   - SecureArc processes + posts PR comment
   - Workflow fails if SecureArc returns any CRITICAL findings

Frontend: /dashboard (enhanced)
- Risk score over time (line chart per project, last 90 days)
- Finding resolution rate (how fast findings get fixed)
- Open findings by category (bar chart)
- PR check history: pass rate trending
- Top 5 most common finding types
- Projects ranked by current risk score

Use Recharts for all charts (already available in Next.js environment).
No external charting CDN needed.

Generate complete, wired, runnable code for all three modules.
Also generate a comprehensive README update covering the full platform.
```

---
---

# Iteration Strategy (How to Use These Prompts)

## The Right Way to Build with Claude

1. **Start a fresh Claude Project** for SecureArc — this gives persistent context
2. **Paste Phase 1 prompt** → generate full code → review → ask Claude to fix issues
3. **Refine iteratively** — don't start over. Say: "Fix the MinIO connection issue in docker-compose.yml" or "The findings page is not showing severity badges — fix it"
4. **Use Claude Code CLI** for large refactors: `claude "refactor the gap_analyzer.py to handle DOCX files better"`
5. **Phase gate**: only move to Phase 2 once Phase 1 checklist is fully green

## Key Iteration Prompts to Keep Handy

```
Fix this error and explain what caused it: [paste error]
```
```
The [feature] is not working as expected. Current behavior: [X]. Expected behavior: [Y]. Fix it.
```
```
Refactor [module] to be more robust — add proper error handling, logging, and input validation.
```
```
Write a complete test suite for [module] using pytest. Cover happy path, error cases, and edge cases.
```
```
Review the entire [module] as a senior security architect. What security issues do you see in the code itself?
```

---

# Local Development Setup (Phase 1 Kickoff)

```bash
# 1. Clone your new repo
git init securearc && cd securearc

# 2. Paste Phase 1 output files into project structure

# 3. Start everything
docker compose up -d

# 4. Initialize DB
docker compose exec backend python -m scripts.init_db

# 5. Open app
open http://localhost:3000

# 6. Upload your first design doc and run analysis
```

## Expected folder structure after Phase 1
```
securearc/
├── docker-compose.yml
├── .env.example
├── README.md
├── frontend/
│   ├── app/
│   │   ├── dashboard/
│   │   ├── projects/
│   │   └── login/
│   ├── components/
│   └── package.json
└── backend/
    ├── main.py
    ├── routers/
    ├── services/
    │   └── gap_analyzer.py
    ├── models/
    └── requirements.txt
```

---

# Security Architecture Principles (Built Into Every Phase)

As a senior security architect, these are non-negotiable for SecureArc itself:

1. **Eat your own dog food** — SecureArc must pass its own security checks
2. **Secrets never in code** — all credentials via environment variables + .env (gitignored)
3. **Auth on every endpoint** — no unauthenticated API routes except /health and /webhooks (signature-verified)
4. **Webhook signature verification** — GitHub HMAC SHA-256 verification before processing any webhook
5. **Role-based access** — ADMIN (manage settings, view all projects) / ANALYST (create projects, upload, view findings) / VIEWER (read-only)
6. **Audit log** — every analysis run, finding state change, and settings change logged to audit_log table
7. **No LLM prompt injection** — user-uploaded document content is clearly delimited from system prompts using XML tags
8. **Rate limiting** — API endpoints rate-limited to prevent abuse
9. **Dependency pinning** — all dependencies pinned to exact versions in requirements.txt and package-lock.json
10. **LLM output validation** — all Claude API responses parsed and validated before storing; never trust raw LLM output as SQL or code

---

*SecureArc Master Build Plan v1.0 — Built for internal AppSec use*
*Phases designed for AI-assisted development with Claude Sonnet 4.6*
