# Project Spec: Clio

> Clio — the Muse of History, telling the story of your code

![](asset://localhost/%2FUsers%2Fmushood%2FDocuments%2Fnotes%2Fassets%2Fclio.png)

## 1. Overview & Vision

**Clio** is a web service that reads any GitHub repository (public, private, or organization), analyzes its structure and code, and produces a beautifully structured, comprehensive `README`. Users can request generations, view, download, and track history of generated READMEs.

Key constraints / features:

- Users authenticate via GitHub.
- Support public, private, and organization GitHub repos.
- Access to repos is managed via a GitHub App (preferred) and fallback OAuth only where necessary.
- Generation is asynchronous; users initiate the task, get notified by email when done, see status in UI.
- No in-repo writing (no automatic push) — only generate, view, download, history, resync / regenerate.

## 2. Tech stack

- `Next.js`.
- `tRPC`.
- `Tailwind`.
- `shadcn/ui`.
- `Prisma` + `Neon PostgreSQL`.
- `Gemini`.

## 3. Name & Branding

- Project name: `Clio` (after Greek Muse of History / storytelling)
- Possible branded variants: ClioGen, ClioDoc, ClioRead, MnemoClio
- Domain / branding considerations: aim for short, memorable, avoid name collisions
- Tagline ideas:
  - `Your code’s storyteller`
  - `From code to story / README`
  - `Illuminate your repository’s narrative`

## 4. Authentication & GitHub Access

### 4.1 Strategy

- Use **NextAuth** with GitHub OAuth for user identity (login).
- Use a **GitHub App** (installable) for repo access (read only).
- Optionally fallback to OAuth `repo` scope for personal repos if the user declines app installation (with clear warning).

### 4.2 Permissions and Scope

- GitHub App requested permissions:
  - Repository → Contents: Read only
  - Repository → Metadata: Read only
- No write permissions (since we won’t push README)
- Setup a setup URL for the GitHub App so that after install it returns the user to Clio
- For OAuth fallback (if used):
  - `repo` scope is broad — must warn user
  - Only use it if they explicitly agree

### 4.3 User Flow (Happy Path)

- User visits Clio → clicks “Sign in with GitHub”
- User authenticated, session established
- Dashboard shows option to install Clio GitHub App — link to `https://github.com/app/<app-slug>/installations/new`
- User installs app (for user account or for an organization)
- In Clio UI, user enters / selects a repo (owner/repo)
- Server checks whether the GitHub App is installed on that repo via GET `/repos/{owner}/{repo}/installation`
  - If not installed: prompt user / org admin to install
- Server uses GitHub App’s installation ID to request an **installation access token** via `@octokit/auth-app`
- Using that token, server fetches repo file tree + contents, then passes data to generation pipeline
- Returns generation request to user (or enqueues)

### 4.4 Edge / Fallback Flows

- If user owns a private personal repo but hasn’t installed the app → prompt them to either install or grant OAuth `repo` scope
- If user is in an org that hasn’t installed the app → show instructions, point to installing by org admin
- If access token / installation token is revoked or expired → detect failure, prompt to re-install or re-authorize
- Handle repos too large or unsupported (binary files, monorepos) gracefully with errors or partial fetch

## 5. Architecture & Components (Auth + Repo Access)

### 5.1 Modules / Layers

- **Auth Layer**: NextAuth (providers, session, user)
- GitHub App / Octokit Layer:
  - Function to check installation ID for a repo
  - Function to obtain installation token
  - Functions to fetch repo metadata, tree listing, file contents
- **API Layer** (Next.js API routes or server actions):
  - Endpoints:
    - `GET /api/auth` (via NextAuth)
    - `POST /api/check-installation`
    - `GET /api/install-url`
    - `POST /api/generate-readme` (enqueue job)
    - `GET /api/job/:id/status` | `GET /api/job/:id/result`
  - **DB / Persistence** (Prisma + Neon Postgres):
    - Tables: `User` , `Installation` , `GenerationJob` , `ReadmeHistory`
    - Store GitHub user info, installation IDs, job metadata, generated README text
  - **Worker / Task Queue**:
    - Job processor picks up GenerationJob tasks
    - Uses GitHub access module to fetch repo
    - Interacts with LLM module to generate README
    - Saves to DB, triggers email notification
  - **Email / Notification Module**:
    - Send email via external service (SendGrid / SES / Mailgun)
    - Template includes job status + link to view result
  - **Frontend / UI:**
    - Login page
    - Dashboard / repo selection
    - Installation prompt / status
    - Job status page
    - History / previous README viewer
    - Download / copy button

# 5. Prisma Schema

```
model User {
   id Int @id @default(autoincrement())
   githubId String @unique
   username String
   avatarUrl String?
   email String?
   installations Installation[]
   jobs GenerationJob[]  
}
```

```
model Installation {
   id Int @id @default(autoincrement())
   installationId Int // GitHub App installation ID
   user User @relation(fields: [userId], references: [id])
   userId Int
   owner String // e.g. GitHub username or org name
   createdAt DateTime @default(now())
   updatedAt DateTime @updatedAt
}
```

```
model GenerationJob {
   id String @id @default(uuid())
   user User @relation(fields: [userId], references: [id])
   userId Int
   repoOwner String
   repoName String
   commitSha String?
   status JobStatus @default(PENDING)
   result String? @db.Text
   errorMessage String? @db.Text
   createdAt DateTime @default(now())
   updatedAt DateTime @updatedAt
}
```

```
enum JobStatus {
   PENDING
   RUNNING
   SUCCEEDED
   FAILED
}
```

# 6. UX / Edge Cases & Error Handling

- If app is not installed for a particular repo: show a clear “Install Clio for this repo / org” prompt
- If user lacks permission (private repo but app not installed): explain the steps
- If API calls fail (404, rate limit, token expired) → catch and show friendly message
- If repo is huge, fallback (partial / chunk) or show “too large” error
- On token revocation / uninstall events, update DB so stale installations are cleaned up
- Use GitHub Webhooks (installation/uninstallation) to sync app install status proactively

&nbsp;