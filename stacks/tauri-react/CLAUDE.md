# CLAUDE.md — Tauri + React (Desktop)
### Forge Stack Rules | Lightweight Cross-Platform Desktop Apps

---

This file extends the universal CLAUDE.md. Read that first.

---

## WHEN TO USE TAURI

- Small, fast desktop apps (Tauri is 5-10x smaller than Electron)
- Apps needing native performance
- Apps with security-sensitive operations (Rust backend is safer than Node)
- Modern cross-platform desktop (Windows, macOS, Linux)

Default over Electron unless the team lacks Rust experience or needs specific Electron ecosystem features.

---

## TECHNOLOGY STANDARDS

### Required
- **Tauri 2.x** (2.0+ is current stable)
- **Rust** (latest stable)
- **React 18+** with TypeScript strict mode
- **Vite** as the build tool
- **Tailwind CSS** for styling

### Frontend State
- **TanStack Query** for data from Rust backend (treat it like any API)
- **Zustand** for client state
- **React Router** for in-app navigation

---

## FOLDER STRUCTURE

```
project-root/
├── CLAUDE.md
├── package.json
├── vite.config.ts
├── src/                           (React frontend)
│   ├── main.tsx
│   ├── App.tsx
│   ├── pages/
│   ├── components/
│   ├── hooks/
│   ├── services/                  (Tauri command wrappers)
│   │   └── tauri.ts
│   ├── stores/
│   └── types/
├── src-tauri/                     (Rust backend)
│   ├── Cargo.toml
│   ├── tauri.conf.json
│   ├── src/
│   │   ├── main.rs
│   │   ├── commands/              (Tauri commands)
│   │   ├── services/
│   │   └── models/
│   └── icons/
└── public/
```

---

## THE FRONTEND

For all React rules, refer to `stacks/react-node/frontend-CLAUDE.md`. Everything there applies here. The only difference is the backend isn't HTTP — it's Tauri commands.

### Tauri Command Wrapper
```typescript
// src/services/tauri.ts
import { invoke } from '@tauri-apps/api/core'

export async function getUsers(): Promise<User[]> {
  return invoke<User[]>('get_users')
}

export async function createUser(input: CreateUserInput): Promise<User> {
  return invoke<User>('create_user', { input })
}
```

- Always wrap `invoke` calls in typed service functions
- Never call `invoke` directly from components
- Handle errors properly — Rust errors come through as thrown errors

---

## THE RUST BACKEND

### Commands
```rust
// src-tauri/src/commands/users.rs
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize, Debug)]
pub struct User {
    pub id: String,
    pub email: String,
    pub name: String,
}

#[derive(Deserialize)]
pub struct CreateUserInput {
    pub email: String,
    pub name: String,
}

#[tauri::command]
pub async fn get_users() -> Result<Vec<User>, String> {
    // implementation
    Ok(vec![])
}

#[tauri::command]
pub async fn create_user(input: CreateUserInput) -> Result<User, String> {
    // validate
    if !input.email.contains('@') {
        return Err("Invalid email".into());
    }
    
    // implementation
    Ok(User {
        id: uuid::Uuid::new_v4().to_string(),
        email: input.email,
        name: input.name,
    })
}
```

### Registering Commands
```rust
// src-tauri/src/main.rs
fn main() {
    tauri::Builder::default()
        .invoke_handler(tauri::generate_handler![
            get_users,
            create_user,
        ])
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

---

## LOCAL DATA STORAGE

### SQLite (Recommended)
Use `sqlx` with SQLite for structured local data:

```toml
# Cargo.toml
[dependencies]
sqlx = { version = "0.7", features = ["runtime-tokio", "sqlite"] }
tokio = { version = "1", features = ["full"] }
```

### Tauri Store Plugin
For simple key-value storage: `tauri-plugin-store`

### Secure Storage
For secrets (API tokens, passwords): `tauri-plugin-stronghold` or OS keychain via `keyring` crate

**Never use localStorage for sensitive data.**

---

## SECURITY

### Tauri Allowlist
- `tauri.conf.json` defines what APIs the frontend can call
- Enable only what you need — every permission is attack surface
- Never enable all permissions

```json
{
  "app": {
    "security": {
      "csp": "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'"
    }
  }
}
```

### Input Validation
- Validate in Rust, never trust frontend
- Use `serde` with `#[serde(deny_unknown_fields)]` to reject extra fields

### Updates
- Use Tauri's built-in updater with signed updates
- Never ship unsigned updates in production

---

## APPLICATION ARCHITECTURE

### Separation
- Frontend: UI, user interactions, view logic
- Rust: file system access, database, OS integrations, heavy computation
- Never duplicate logic between them — decide where each piece lives

### Background Tasks
- Use Tokio for async Rust operations
- Emit events to the frontend from Rust for long-running tasks:
```rust
use tauri::Emitter;

app_handle.emit("task-progress", progress)?;
```

---

## TESTING

### Frontend
Same as React frontend rules — Vitest + React Testing Library + Playwright

### Rust
- `cargo test` for unit tests
- Integration tests in `src-tauri/tests/`
- Mock Tauri APIs when testing business logic

---

## BUILD AND DISTRIBUTION

```bash
# Development
npm run tauri dev

# Production build
npm run tauri build
```

Output: Platform-specific installers in `src-tauri/target/release/bundle/`
- Windows: `.msi`, `.exe`
- macOS: `.dmg`, `.app`
- Linux: `.deb`, `.AppImage`, `.rpm`

### Code Signing
- macOS: Apple Developer ID certificate
- Windows: Code signing certificate (EV for immediate SmartScreen trust)
- Configure in `tauri.conf.json` under `bundle.identifier` and `updater.pubkey`

---

## SPECIFIC DO NOTS

- Never use `dangerousUseHttpScheme`
- Never disable CSP
- Never expose file system commands without path validation
- Never ship unsigned binaries to users
- Never skip the Tauri allowlist configuration
- Never use `unsafe` Rust without thorough justification

---

## SESSION COMPLETION CHECKLIST — TAURI SPECIFIC

- [ ] `npm run tauri dev` launches without errors
- [ ] `cargo check` in `src-tauri/` shows zero warnings
- [ ] `cargo clippy` shows zero warnings
- [ ] Rust tests pass (`cargo test`)
- [ ] Frontend tests pass
- [ ] App builds for target platforms
- [ ] CSP configured properly
- [ ] Only necessary Tauri permissions enabled
- [ ] Error handling on both sides of the bridge

---

*This file is part of Forge by Sultan Al-Muqimi / NQTH LLC.*
