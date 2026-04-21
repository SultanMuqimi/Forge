# CLAUDE.md — Electron + React (Desktop)
### Forge Stack Rules | Cross-Platform Desktop Apps (Electron)

---

This file extends the universal CLAUDE.md. Read that first.

---

## WHEN TO USE ELECTRON OVER TAURI

- Team has no Rust experience
- Heavy use of Node.js ecosystem libraries
- Need specific Electron ecosystem features (some native modules)
- Existing web app being wrapped for desktop

**For most new projects, prefer Tauri** — smaller binaries, better security, less memory.

---

## TECHNOLOGY STANDARDS

### Required
- **Electron** (latest stable)
- **Electron Forge** or **electron-vite** as the tooling
- **React 18+** with TypeScript strict mode
- **Vite** as the build tool
- **Tailwind CSS** for styling
- Context isolation ENABLED, node integration DISABLED in renderer

### Frontend State
Same as React+Node frontend — TanStack Query, Zustand, React Router.

---

## FOLDER STRUCTURE (Electron Forge with Vite)

```
project-root/
├── CLAUDE.md
├── package.json
├── forge.config.ts
├── vite.main.config.ts
├── vite.preload.config.ts
├── vite.renderer.config.ts
├── tsconfig.json
└── src/
    ├── main/                      (main process)
    │   ├── index.ts               (entry point)
    │   ├── window.ts              (window management)
    │   ├── ipc/                   (IPC handlers by feature)
    │   ├── services/              (business logic)
    │   └── database/              (if using SQLite)
    ├── preload/                   (preload scripts)
    │   └── index.ts               (exposes safe APIs to renderer)
    ├── renderer/                  (React frontend)
    │   ├── index.html
    │   ├── main.tsx
    │   ├── App.tsx
    │   ├── pages/
    │   ├── components/
    │   ├── hooks/
    │   ├── services/              (IPC wrappers)
    │   └── types/
    └── shared/                    (types shared between main and renderer)
        └── ipc-types.ts
```

---

## SECURITY — NON-NEGOTIABLE

### Every Electron App Must Have
```typescript
new BrowserWindow({
  webPreferences: {
    contextIsolation: true,         // REQUIRED
    nodeIntegration: false,          // REQUIRED
    sandbox: true,                   // STRONGLY RECOMMENDED
    webSecurity: true,               // REQUIRED
    preload: path.join(__dirname, '../preload/index.js'),
  },
})
```

**Never disable contextIsolation.** **Never enable nodeIntegration in the renderer.**

### Content Security Policy
Set in `index.html`:
```html
<meta http-equiv="Content-Security-Policy" content="default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'">
```

### Safe URLs
- Validate every URL before `shell.openExternal`
- Never navigate the renderer to arbitrary URLs
- Use `will-navigate` handler to block unexpected navigation

---

## IPC PATTERN — TYPE-SAFE

### Shared Types
```typescript
// src/shared/ipc-types.ts
export interface IpcApi {
  getUsers: () => Promise<User[]>
  createUser: (input: CreateUserInput) => Promise<User>
}

export type IpcChannel = keyof IpcApi
```

### Preload (Exposes Safe API)
```typescript
// src/preload/index.ts
import { contextBridge, ipcRenderer } from 'electron'
import type { IpcApi } from '@shared/ipc-types'

const api: IpcApi = {
  getUsers: () => ipcRenderer.invoke('users:get-all'),
  createUser: (input) => ipcRenderer.invoke('users:create', input),
}

contextBridge.exposeInMainWorld('api', api)
```

### Type Declaration for Renderer
```typescript
// src/renderer/types/global.d.ts
import type { IpcApi } from '@shared/ipc-types'

declare global {
  interface Window {
    api: IpcApi
  }
}
```

### Main Process Handler
```typescript
// src/main/ipc/users.ts
import { ipcMain } from 'electron'
import { usersService } from '../services/users.service'

export function registerUsersHandlers() {
  ipcMain.handle('users:get-all', async () => {
    return usersService.getAll()
  })
  
  ipcMain.handle('users:create', async (_event, input) => {
    // VALIDATE input — never trust renderer
    return usersService.create(input)
  })
}
```

### Renderer Usage
```typescript
// src/renderer/services/users.ts
export const usersService = {
  getAll: () => window.api.getUsers(),
  create: (input: CreateUserInput) => window.api.createUser(input),
}
```

---

## INPUT VALIDATION — MAIN PROCESS

**Every IPC handler validates inputs.** Never trust the renderer — a compromised renderer could send anything.

```typescript
import { z } from 'zod'

const createUserSchema = z.object({
  email: z.string().email(),
  name: z.string().min(1).max(100),
})

ipcMain.handle('users:create', async (_event, input) => {
  const parsed = createUserSchema.parse(input)
  return usersService.create(parsed)
})
```

---

## LOCAL DATA STORAGE

### SQLite (Recommended for Structured Data)
- **better-sqlite3** for synchronous API (simpler)
- **sqlite3** for async API (better for concurrent operations)
- Use migrations (Knex, Drizzle, or manual)

### Settings/Preferences
- **electron-store** for user preferences (JSON-based)
- Never store sensitive data here

### Secrets
- **keytar** (OS keychain integration) — deprecated but still works
- **safeStorage** API (built into Electron) — modern approach

---

## AUTO-UPDATES

- Use **electron-updater** with signed releases
- Host updates on a secure server (S3, GitHub Releases)
- Never ship unsigned updates

---

## WINDOWS AND MENUS

### Main Window
- Remember position and size in user preferences
- Handle `closed` and `close` events properly
- Show after `ready-to-show` event (prevents white flash)

### Menu
- Use `Menu.setApplicationMenu(menu)` with platform-aware menus
- On macOS: proper app menu structure (first submenu is the app name)
- Use role-based menu items where possible (`role: 'quit'`, `role: 'undo'`)

---

## TESTING

### Stack
- **Vitest** for unit tests
- **Playwright** for E2E tests (Playwright supports Electron)
- Test the main process and renderer separately

### Main Process Tests
- Mock `electron` APIs
- Test services and IPC handlers in isolation

---

## BUILD AND DISTRIBUTION

### Electron Forge
```bash
# Dev
npm run start

# Package
npm run package

# Make installers
npm run make
```

Output formats (configure in `forge.config.ts`):
- Windows: Squirrel.Windows, MSI
- macOS: DMG, ZIP
- Linux: DEB, RPM, AppImage

### Code Signing
- macOS: Apple Developer ID + notarization (required for distribution)
- Windows: Code signing certificate
- Linux: GPG signing optional

---

## PERFORMANCE

- Don't load React DevTools in production
- Use production React build (`npm run build`)
- Profile main process CPU usage — it blocks everything
- Heavy computation goes in worker threads or utility processes

---

## ELECTRON SPECIFIC DO NOTS

- Never enable `nodeIntegration` in the renderer
- Never disable `contextIsolation`
- Never use `remote` module (it's deprecated and insecure)
- Never use `eval()` in either process
- Never load remote content in `BrowserWindow` without strict CSP
- Never expose `ipcRenderer` directly to the renderer — always go through contextBridge
- Never use `enableRemoteModule` (deprecated)
- Never ship with developer tools open in production
- Never skip input validation in IPC handlers

---

## SESSION COMPLETION CHECKLIST — ELECTRON SPECIFIC

- [ ] `contextIsolation: true` and `nodeIntegration: false` in all windows
- [ ] CSP configured in renderer HTML
- [ ] All IPC handlers validate inputs
- [ ] Preload script uses `contextBridge` only (no direct exposure)
- [ ] Type-safe IPC with shared types
- [ ] Dev and production builds work
- [ ] App signs and packages for target platforms
- [ ] No remote module usage
- [ ] Auto-updater configured (if applicable)

---

*This file is part of Forge by Sultan Al-Muqimi / NQTH LLC.*
