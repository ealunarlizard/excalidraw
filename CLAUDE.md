# CLAUDE.md

## Project Structure

Excalidraw is a **monorepo** with a clear separation between the core library and the application:

- **`packages/excalidraw/`** - Main React component library published to npm as `@excalidraw/excalidraw`
- **`excalidraw-app/`** - Full-featured web application (excalidraw.com) that uses the library
- **`packages/`** - Core packages: `@excalidraw/common`, `@excalidraw/element`, `@excalidraw/math`, `@excalidraw/utils`
- **`examples/`** - Integration examples (NextJS, browser script)

## Development Workflow

1. **Package Development**: Work in `packages/*` for editor features
2. **App Development**: Work in `excalidraw-app/` for app-specific features
3. **Testing**: Always run `yarn test:update` before committing
4. **Type Safety**: Use `yarn test:typecheck` to verify TypeScript

## Development Commands

```bash
yarn test:typecheck  # TypeScript type checking
yarn test:update     # Run all tests (with snapshot updates)
yarn fix             # Auto-fix formatting and linting issues
```

## Architecture Notes

### Package System

- Uses Yarn workspaces for monorepo management
- Internal packages use path aliases (see `vitest.config.mts`)
- Build system uses esbuild for packages, Vite for the app
- TypeScript throughout with strict configuration

---

## Self-Hosting Guide

> Goal: deploy an internal instance that matches excalidraw.com feature-for-feature — real-time collaboration (room sharing) + Google account login.

### Overview: Three Services Required

The full-featured app requires **three separate deployments**. Only service 1 lives in this repo.

| Service | Role | Repo |
|---------|------|------|
| **1. Frontend** (this repo) | React SPA served by Nginx | `ll-excalidraw` |
| **2. Collab server** | Socket.io WebSocket server for real-time sync | [`excalidraw/excalidraw-room`](https://github.com/excalidraw/excalidraw-room) |
| **3. Firebase project** | Persistent encrypted room storage + file uploads | Firebase (or self-hosted alternative) |

> **Important:** The official docs note: *"self-hosting does not support sharing or collaboration features"* out of the box — you need all three services for that. See [dev-docs/docs/introduction/development.mdx](dev-docs/docs/introduction/development.mdx).

### How Real-Time Collaboration Works

1. User clicks "Start Collaboration"
2. Frontend generates a random `roomId` + AES encryption key
3. Connects to the **collab server** via `socket.io-client` → `VITE_APP_WS_SERVER_URL`
4. Encrypted scene deltas are broadcast to all peers through the socket server
5. Every ~15 s the encrypted room state is also saved to **Firebase Firestore** (for new joiners to load history)
6. Images are saved to **Firebase Storage**
7. The share link format is `#room={roomId},{encryptionKey}` — the server never sees plaintext

Key files:
- [excalidraw-app/collab/Collab.tsx](excalidraw-app/collab/Collab.tsx) — core collab logic, socket setup
- [excalidraw-app/collab/Portal.tsx](excalidraw-app/collab/Portal.tsx) — socket event handling, encrypted broadcast
- [excalidraw-app/data/firebase.ts](excalidraw-app/data/firebase.ts) — Firestore/Storage read-write

### Authentication: Current State

**The codebase has zero user authentication.** Room access is purely URL-based — anyone with the link can join. The Firebase security rules in [firebase-project/firestore.rules](firebase-project/firestore.rules) are completely open (`allow read, write: if true`).

The only "auth" concept is:
- A localStorage username (display name only, not verified)
- A `excplus-auth` cookie that gates Excalidraw+ premium export features

To add Google OAuth you must implement it yourself (see section below).

### Environment Variables

All config is baked in at **build time** via Vite env vars. See [.env.production](.env.production) and [.env.development](.env.development).

| Variable | Purpose | Example |
|----------|---------|---------|
| `VITE_APP_WS_SERVER_URL` | URL of your collab server | `https://collab.internal.example.com` |
| `VITE_APP_FIREBASE_CONFIG` | JSON blob of Firebase project config | `{"apiKey":"...","projectId":"..."}` |
| `VITE_APP_BACKEND_V2_GET_URL` | URL for loading saved drawings (link sharing) | `https://json.excalidraw.com/api/v2/` |
| `VITE_APP_BACKEND_V2_POST_URL` | URL for saving drawings to a shareable link | `https://json.excalidraw.com/api/v2/post/` |
| `VITE_APP_LIBRARY_URL` | Element library source | Can keep pointing to `libraries.excalidraw.com` |
| `VITE_APP_PLUS_APP` | Excalidraw+ app URL (for premium prompts) | Leave blank or remove to suppress |

> The `BACKEND_V2` endpoints are for the "Share a link" feature (not collaboration). They point to `json.excalidraw.com` — an additional service not included here. You can skip this or run a compatible backend.

### Option A — Minimal (drawing only, no collab)

No collab server or Firebase needed. Just build and serve the frontend.

```bash
# Build
docker build -t excalidraw .

# Run (all collab UI will be present but non-functional without WS_SERVER_URL)
docker run -p 3000:80 excalidraw
```

### Option B — Full Collaboration (no auth)

Matches excalidraw.com collaboration without login gating.

**Step 1 — Set up Firebase**
1. Create a Firebase project at console.firebase.google.com
2. Enable **Firestore** and **Storage**
3. Copy [firebase-project/firestore.rules](firebase-project/firestore.rules) and [firebase-project/storage.rules](firebase-project/storage.rules) to your project
4. Note your project's config JSON (from Project Settings → Your apps)

**Step 2 — Deploy the collab server**
```bash
git clone https://github.com/excalidraw/excalidraw-room
cd excalidraw-room
docker build -t excalidraw-room .
docker run -d -p 3002:3002 --name collab excalidraw-room
```

**Step 3 — Build and deploy the frontend**

Create `.env.local` at the repo root (never commit this):
```env
VITE_APP_WS_SERVER_URL=https://collab.internal.example.com
VITE_APP_FIREBASE_CONFIG='{"apiKey":"YOUR_KEY","authDomain":"YOUR_PROJECT.firebaseapp.com","projectId":"YOUR_PROJECT","storageBucket":"YOUR_PROJECT.appspot.com","messagingSenderId":"YOUR_ID","appId":"YOUR_APP_ID"}'
```

Then build:
```bash
docker build -t excalidraw-internal \
  --build-arg NODE_ENV=production .
docker run -d -p 3000:80 excalidraw-internal
```

### Option C — Full Collaboration + Google OAuth (recommended for internal infra)

This requires modifying the app. The collab server and Firebase stay the same as Option B.

**Architecture addition:**
- Firebase Authentication (Google provider) — sits alongside Firestore/Storage in the same Firebase project
- A sign-in gate component in the frontend
- Firebase security rules updated to require `auth.uid`
- The collab server optionally validates Firebase ID tokens (JWTs) on socket connection

**Implementation steps:**

1. **Enable Firebase Auth** in your Firebase project (Authentication → Sign-in method → Google)

2. **Add Firebase Auth to the frontend** — in [excalidraw-app/data/firebase.ts](excalidraw-app/data/firebase.ts):
   ```ts
   import { getAuth, GoogleAuthProvider, signInWithPopup } from "firebase/auth";
   export const auth = getAuth(firebaseApp);
   export const signInWithGoogle = () => signInWithPopup(auth, new GoogleAuthProvider());
   ```

3. **Add a sign-in gate** — wrap [excalidraw-app/App.tsx](excalidraw-app/App.tsx) with an auth check that calls `signInWithGoogle()` when no user is present.

4. **Tighten Firebase security rules** in [firebase-project/firestore.rules](firebase-project/firestore.rules):
   ```
   rules_version = '2';
   service cloud.firestore {
     match /databases/{database}/documents {
       match /{document=**} {
         allow read, write: if request.auth != null;
       }
     }
   }
   ```
   Apply the same to [firebase-project/storage.rules](firebase-project/storage.rules).

5. **(Optional) Validate tokens on the collab server** — in `excalidraw-room`, verify the Firebase ID token passed via socket handshake auth to prevent unauthenticated socket connections.

6. **Restrict to your domain** — in Firebase Console → Authentication → Settings → Authorized domains, add only your internal domain.

### Docker Compose (all services)

A `docker-compose.yml` for the complete stack:

```yaml
services:
  excalidraw:
    build: .
    ports:
      - "3000:80"
    env_file: .env.local      # contains VITE_APP_WS_SERVER_URL + VITE_APP_FIREBASE_CONFIG

  collab:
    image: excalidraw-room    # build from excalidraw/excalidraw-room
    ports:
      - "3002:3002"
    environment:
      - PORT=3002
```

### Effort Estimate

| Goal | Effort |
|------|--------|
| Static drawing app only | < 1 hour (just `docker build`) |
| Collaboration without auth | 2–4 hours (Firebase project + collab server + env config) |
| Collaboration + Google OAuth | 1–2 days (Firebase Auth integration + sign-in UI + security rules) |

### Known Limitations of Self-Hosting

- **No `excalidraw-json` backend included** — "Share link" (non-collab) won't work unless you run a compatible backend or leave those env vars pointing at the official `json.excalidraw.com`
- **No Excalidraw+ features** — AI diagramming, export to PNG/SVG in cloud, team workspaces are SaaS-only
- **No element library sync** — can keep pointing at `libraries.excalidraw.com` unless you need an internal mirror
