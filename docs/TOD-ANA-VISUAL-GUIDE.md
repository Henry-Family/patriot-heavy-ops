# Tod & Ana Visual Setup Guide

> **Visual walkthrough with diagrams and examples**

## �� The Big Picture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           YOUR DEVELOPMENT WORKFLOW                      │
└─────────────────────────────────────────────────────────────────────────┘

    You push code                CI tests fail              Bugbot reviews code
         │                            │                              │
         │                            │                              │
         ▼                            ▼                              ▼
    ┌─────────┐                 ┌─────────┐                    ┌─────────┐
    │ GitHub  │                 │ GitHub  │                    │ Cursor  │
    │ Actions │                 │ Actions │                    │ Bugbot  │
    └────┬────┘                 └────┬────┘                    └────┬────┘
         │                           │                              │
         └───────────────────────────┴──────────────────────────────┘
                                     │
                                     │ All trigger...
                                     │
                                     ▼
                            ┌──────────────┐
                            │     ANA      │ ← GitHub Actions Workflow
                            │  (Analyzer)  │   Monitors & Analyzes
                            └──────┬───────┘
                                   │
                                   │ 1. Analyzes logs/comments
                                   │ 2. Extracts actionable items
                                   │ 3. Signs with HMAC-SHA256
                                   │ 4. Sends webhook POST
                                   │
                                   ▼
                            ┌──────────────┐
                            │     TOD      │ ← Cursor Background Agent
                            │  (Receiver)  │   Webhook Server
                            └──────┬───────┘
                                   │
                                   │ 1. Validates signature
                                   │ 2. Checks timestamp
                                   │ 3. Transforms to TODOs
                                   │ 4. Calls Cursor API
                                   │
                                   ▼
                            ┌──────────────┐
                            │    CURSOR    │ ← Your IDE
                            │   TODO List  │   Native Integration
                            └──────────────┘
                                   │
                                   ▼
                              You see TODOs!
                              Fix issues quickly!
```

---

## 🔑 The Three Keys Explained

```
╔═══════════════════════════════════════════════════════════════════════════╗
║                       KEY #1: ANA_WEBHOOK_SECRET                          ║
╚═══════════════════════════════════════════════════════════════════════════╝

What:     a3f7b2c9d8e1f6a4b7c2d9e3f8a1b6c4d7e2f9a3b8c1d6e4f7a2b9c3d8e1f6a4
Where:    [GitHub Secrets] + [Tod .env.local]
Purpose:  Cryptographic signature for webhook authentication
Critical: ⚠️  MUST BE IDENTICAL IN BOTH PLACES ⚠️

┌─────────────────────────────────────┐    ┌─────────────────────────────────────┐
│        GitHub Repository            │    │      Tod Server (.env.local)        │
│  Settings → Secrets → Actions       │    │                                     │
│                                     │    │                                     │
│  ANA_WEBHOOK_SECRET =               │═══▶│  ANA_WEBHOOK_SECRET =               │
│    "a3f7b2c9d8e1...f6a4"            │    │    "a3f7b2c9d8e1...f6a4"            │
│                                     │    │                                     │
│  ✅ Must match exactly! ✅           │    │  ✅ Must match exactly! ✅           │
└─────────────────────────────────────┘    └─────────────────────────────────────┘


╔═══════════════════════════════════════════════════════════════════════════╗
║                     KEY #2: TOD_WEBHOOK_ENDPOINT                          ║
╚═══════════════════════════════════════════════════════════════════════════╝

What:     https://abc123.ngrok.io/webhook/ana-failures
Where:    [GitHub Secrets ONLY]
Purpose:  Tells Ana where to send webhook requests

Development:
┌─────────────────────────────────────┐
│     Your Computer (localhost)       │
│                                     │
│   Tod Server → Port 3001            │
│   http://localhost:3001             │
└────────────────┬────────────────────┘
                 │
                 │ Exposed via ngrok
                 │
                 ▼
┌─────────────────────────────────────┐    ┌─────────────────────────────────────┐
│         Ngrok Tunnel                │    │       GitHub Repository             │
│  https://abc123.ngrok.io            │◀───│  Settings → Secrets → Actions       │
│                                     │    │                                     │
│  /webhook/ana-failures ─┐           │    │  TOD_WEBHOOK_ENDPOINT =             │
│                         │           │    │    "https://abc123.ngrok.io         │
│  Internet-accessible ✅  │           │    │     /webhook/ana-failures"          │
└─────────────────────────┼───────────┘    └─────────────────────────────────────┘
                          │
                          │ Ana sends POST here
                          │

Production:
┌─────────────────────────────────────┐    ┌─────────────────────────────────────┐
│    Vercel/Railway/etc               │    │       GitHub Repository             │
│  https://your-app.vercel.app        │◀───│                                     │
│                                     │    │  TOD_WEBHOOK_ENDPOINT =             │
│  /api/webhooks/ana-failures         │    │    "https://your-app.vercel.app     │
│                                     │    │     /api/webhooks/ana-failures"     │
└─────────────────────────────────────┘    └─────────────────────────────────────┘


╔═══════════════════════════════════════════════════════════════════════════╗
║                         KEY #3: ORG_PAT                                   ║
╚═══════════════════════════════════════════════════════════════════════════╝

What:     github_pat_11XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
Where:    [GitHub Secrets]
Purpose:  Allows Ana to read GitHub data (already set up)
Status:   ✅ You already have this!

┌─────────────────────────────────────┐
│       GitHub Repository             │
│  Settings → Secrets → Actions       │
│                                     │
│  ORG_PAT = "github_pat_11XXX..."    │
│                                     │
│  Permissions:                       │
│    ✅ repo (read)                    │
│    ✅ workflow (read)                │
│    ✅ pull_request (read)            │
└─────────────────────────────────────┘
```

---

## 🔐 Authentication Flow Diagram

```
STEP 1: Ana Creates Payload
┌────────────────────────────────────────────┐
│   {                                        │
│     "summary": "CI failed with 3 errors",  │
│     "analysisDate": "2025-09-30T12:00Z",   │
│     "prNumber": 284,                       │
│     "failures": [                          │
│       {                                    │
│         "id": "ci-12345",                  │
│         "content": "Fix TypeScript error", │
│         "priority": "high",                │
│         "files": ["user-auth-form.tsx"]    │
│       }                                    │
│     ]                                      │
│   }                                        │
└────────────────────────────────────────────┘

STEP 2: Ana Signs Payload
┌────────────────────────────────────────────┐
│  const signature = crypto                  │
│    .createHmac('sha256', SECRET)           │
│    .update(JSON.stringify(payload))        │
│    .digest('hex')                          │
│                                            │
│  Result: "7a8b9c0d1e2f3a4b..."             │
└────────────────────────────────────────────┘

STEP 3: Ana Sends HTTP Request
┌────────────────────────────────────────────┐
│  POST https://abc123.ngrok.io/             │
│       webhook/ana-failures                 │
│                                            │
│  Headers:                                  │
│    Content-Type: application/json          │
│    X-Ana-Signature: sha256=7a8b9c0d...     │
│    X-Ana-Timestamp: 2025-09-30T12:00Z      │
│                                            │
│  Body:                                     │
│    { "summary": "...", "failures": [...] } │
└────────────────────────────────────────────┘
             │
             │ HTTPS (encrypted)
             │
             ▼
STEP 4: Tod Receives Request
┌────────────────────────────────────────────┐
│  1. Extract signature from header          │
│  2. Extract timestamp from header          │
│  3. Read request body as string            │
└────────────────────────────────────────────┘

STEP 5: Tod Recomputes Signature
┌────────────────────────────────────────────┐
│  const expected = crypto                   │
│    .createHmac('sha256', SECRET)           │
│    .update(requestBody)                    │
│    .digest('hex')                          │
└────────────────────────────────────────────┘

STEP 6: Tod Validates
┌────────────────────────────────────────────┐
│  ✓ Signature matches? (timing-safe)        │
│  ✓ Timestamp < 5 minutes old?              │
│                                            │
│  If BOTH pass → ✅ VALID REQUEST            │
│  If ANY fails  → ❌ REJECT REQUEST          │
└────────────────────────────────────────────┘

STEP 7: Tod Creates TODOs
┌────────────────────────────────────────────┐
│  1. Transform failures to TODO format      │
│  2. Call Cursor's todo_write() API         │
│  3. TODOs appear in Cursor UI              │
└────────────────────────────────────────────┘
             │
             ▼
        ✨ SUCCESS! ✨
```

---

## 📋 Setup Steps Visual

```
STEP 1: Generate Secret
┌─────────────────────────────────────────────┐
│  $ node -e "console.log(                    │
│      require('crypto')                      │
│      .randomBytes(32)                       │
│      .toString('hex'))"                     │
│                                             │
│  Output:                                    │
│  a3f7b2c9d8e1f6a4b7c2d9e3f8a1b6c4d7e2...   │
│                                             │
│  📋 COPY THIS! You'll need it 3 times      │
└─────────────────────────────────────────────┘

STEP 2: Add to GitHub Secrets
┌─────────────────────────────────────────────┐
│  Browser: github.com/USER/REPO/settings/   │
│           secrets/actions                   │
│                                             │
│  Click: "New repository secret"             │
│                                             │
│  Name:  ANA_WEBHOOK_SECRET                  │
│  Value: a3f7b2c9d8e1f6a4b7c2...             │
│                                             │
│  Click: "Add secret"                        │
└─────────────────────────────────────────────┘

STEP 3: Add to Local Environment
┌─────────────────────────────────────────────┐
│  File: .env.local                           │
│                                             │
│  ANA_WEBHOOK_SECRET="a3f7b2c9d8e1..."       │
│  TOD_WEBHOOK_ENDPOINT="http://localhost:    │
│                        3001/webhook/        │
│                        ana-failures"        │
│  NODE_ENV="development"                     │
└─────────────────────────────────────────────┘

STEP 4: Start Tod & Ngrok
┌─────────────────────────────────────────────┐
│  Terminal 1:                                │
│  $ npm run tod:webhook                      │
│                                             │
│  ✅ Tod running on http://localhost:3001    │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│  Terminal 2:                                │
│  $ ngrok http 3001                          │
│                                             │
│  ✅ Forwarding:                             │
│     https://abc123.ngrok.io → :3001         │
│                                             │
│  📋 COPY THIS URL!                          │
└─────────────────────────────────────────────┘

STEP 5: Update GitHub Secret
┌─────────────────────────────────────────────┐
│  $ echo "https://abc123.ngrok.io/           │
│          webhook/ana-failures" |            │
│    gh secret set TOD_WEBHOOK_ENDPOINT       │
│                                             │
│  ✅ Secret TOD_WEBHOOK_ENDPOINT created     │
└─────────────────────────────────────────────┘

STEP 6: Test It!
┌─────────────────────────────────────────────┐
│  $ npx tsx scripts/test-tod-integration.ts  │
│                                             │
│  🧪 Testing Tod Webhook Integration         │
│  1️⃣ ✅ Health check passed                  │
│  2️⃣ ✅ Webhook processed successfully        │
│  3️⃣ ✅ Test TODO created                     │
│                                             │
│  🎉 All tests passed!                       │
└─────────────────────────────────────────────┘
```

---

## 🐛 Troubleshooting Visual

```
ERROR: "Invalid signature"
┌──────────────────────────────────────────────────────────────┐
│  Problem: Secrets don't match                                │
│                                                              │
│  GitHub:      ANA_WEBHOOK_SECRET = "a3f7b2c9..."            │
│  Tod .env:    ANA_WEBHOOK_SECRET = "DIFFERENT!"  ❌          │
│                                                              │
│  Solution:    Make them identical!                           │
│                                                              │
│  Check:       $ cat .env.local | grep ANA_WEBHOOK_SECRET    │
│               $ gh secret list                              │
└──────────────────────────────────────────────────────────────┘

ERROR: "Connection refused"
┌──────────────────────────────────────────────────────────────┐
│  Problem: Tod is not running                                 │
│                                                              │
│  GitHub → Ana → Sends POST → Tod (OFFLINE) ❌                │
│                                                              │
│  Solution:    Start Tod server                               │
│                                                              │
│  Check:       $ curl http://localhost:3001/health           │
│  Start:       $ npm run tod:webhook                         │
└──────────────────────────────────────────────────────────────┘

ERROR: "404 Not Found"
┌──────────────────────────────────────────────────────────────┐
│  Problem: Wrong endpoint URL                                 │
│                                                              │
│  GitHub Secret: https://abc.ngrok.io/webhook  ❌             │
│  Correct:       https://abc.ngrok.io/webhook/ana-failures ✅ │
│                                     ^^^^^^^^^^^^             │
│                                     Missing path!            │
│                                                              │
│  Solution:    Update GitHub secret with full path            │
└──────────────────────────────────────────────────────────────┘
```

---

## ✅ Success Checklist Visual

```
┌─────────────────────────────────────────────────────────────┐
│                    SETUP CHECKLIST                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  [ ] Generated webhook secret (32+ bytes)                  │
│      └─ Command: node -e "console.log(...)"                │
│                                                             │
│  [ ] Added ANA_WEBHOOK_SECRET to GitHub                    │
│      └─ Location: Repo → Settings → Secrets → Actions      │
│                                                             │
│  [ ] Added ANA_WEBHOOK_SECRET to .env.local                │
│      └─ Verify: cat .env.local | grep ANA                  │
│                                                             │
│  [ ] Started Tod server on port 3001                       │
│      └─ Command: npm run tod:webhook                       │
│                                                             │
│  [ ] Exposed Tod via ngrok                                 │
│      └─ Command: ngrok http 3001                           │
│                                                             │
│  [ ] Added TOD_WEBHOOK_ENDPOINT to GitHub                  │
│      └─ Value: https://abc123.ngrok.io/webhook/ana-fail... │
│                                                             │
│  [ ] Health check passes                                   │
│      └─ Test: curl http://localhost:3001/health            │
│                                                             │
│  [ ] Integration test passes                               │
│      └─ Run: npx tsx scripts/test-tod-integration.ts       │
│                                                             │
│  [ ] Triggered Ana workflow from GitHub                    │
│      └─ Method: Push failing tests or wait for Bugbot      │
│                                                             │
│  [ ] TODOs appeared in Cursor                              │
│      └─ Check: Cursor UI TODO list                         │
│                                                             │
│  ✅ ALL DONE! Tod & Ana are working! 🎉                     │
└─────────────────────────────────────────────────────────────┘
```

---

## 📚 Documentation Map

```
                     ┌───────────────────────┐
                     │  TOD-ANA-DOCS-INDEX   │ ← START HERE
                     │  (Navigation Hub)     │
                     └───────────┬───────────┘
                                 │
                ┌────────────────┼────────────────┐
                │                │                │
                ▼                ▼                ▼
    ┌──────────────────┐ ┌─────────────┐ ┌──────────────┐
    │   QUICK START    │ │ COPY-PASTE  │ │   COMPLETE   │
    │   5 min setup    │ │  Commands   │ │    GUIDE     │
    │   First timers   │ │  Terminal   │ │  Production  │
    └──────────────────┘ └─────────────┘ └──────────────┘
                                 │
                ┌────────────────┼────────────────┐
                │                │                │
                ▼                ▼                ▼
    ┌──────────────────┐ ┌─────────────┐ ┌──────────────┐
    │  AUTH FLOW       │ │ CHEAT SHEET │ │    VISUAL    │
    │  Security        │ │  Reference  │ │    GUIDE     │
    │  Deep dive       │ │  One page   │ │  Diagrams    │
    └──────────────────┘ └─────────────┘ └──────────────┘
```

---

**Last Updated**: September 30, 2025  
**Version**: 1.0.0
