# oneKey

> **Status: Architecture phase — implementation not yet started.**

A zero-knowledge encrypted SaaS platform for password and document management. The server never holds plaintext — not during transit, not at rest, not even under breach.

---

## What is oneKey?

oneKey is a full-stack security platform built on a single guarantee: **your data is yours alone**. All encryption and decryption happens exclusively on the client. By the time any data reaches the server, it is already opaque ciphertext. The server stores blobs it cannot read, verifies auth tags it cannot use, and logs metadata it cannot correlate back to plaintext.

---

## Architecture Overview

### Zero-Knowledge Encryption Model

```
Client (trusted)                        Server (zero-knowledge)
─────────────────────────               ──────────────────────────────
Master key derivation                   Auth service (SRP)
  └─ PBKDF2 / Argon2id · salt/user        └─ Stores verifier, never password

Auth sub-key (HKDF-SHA256)    ──────►  Encrypted blob store
  └─ Sent as SRP token                    └─ Ciphertext + IV + auth tag only

Encryption sub-key (AES-256-GCM) ───►  Encrypted DEK store
  └─ IV per operation                     └─ Wrapped DEKs, opaque to server

Document encrypt              ──────►  Audit log (metadata only)
  └─ plaintext → ciphertext + IV + tag    └─ Timestamps, user IDs, op codes

Wrapped key vault                       HSM / KMS
  └─ DEKs wrapped by KEK client-side      └─ Seals wrapped DEKs at rest
```

All client↔server communication travels over **TLS + ciphertext only**. The server is architecturally incapable of reconstructing plaintext.

---

### Upload Flow

| Step | Action | Detail |
|------|--------|--------|
| 1 | Generate random DEK | `crypto.getRandomValues(32 bytes)` |
| 2 | Encrypt payload | AES-256-GCM · random 12-byte IV |
| 3 | Wrap DEK with KEK | `AES-256-GCM(KEK, DEK)` |
| 4 | POST over TLS | `{ ciphertext, IV, tag, wrappedDEK }` |
| 5 | Server stores blobs | Auth tag verified · no decryption |

> **If server is breached:** Attacker gets ciphertext only. No keys → no plaintext.

### Download Flow

| Step | Action | Detail |
|------|--------|--------|
| 1 | GET request (authenticated) | Server returns ciphertext + wrappedDEK |
| 2 | Unwrap DEK | `AES-256-GCM decrypt(KEK, wrappedDEK)` |
| 3 | Decrypt ciphertext | Verify GCM tag · recover plaintext |
| 4 | Render plaintext | In-memory only · never re-uploaded |

---

## Tech Stack

| Layer | Technology |
|-------|------------|
| Frontend | Next.js (App Router) |
| Backend | Next.js API routes / Node.js |
| Database | PostgreSQL with Row-Level Security (RLS) |
| Storage | AWS S3 (encrypted blobs) |
| Compute | AWS EC2 |
| Database hosting | AWS RDS |
| Key management | AWS KMS / HSM |
| CI/CD | GitHub Actions · Docker containers |

---

## Cryptographic Primitives

| Primitive | Usage |
|-----------|-------|
| `Argon2id` / `PBKDF2` | Master key derivation from user password |
| `HKDF-SHA256` | Sub-key derivation (auth key, encryption key) |
| `AES-256-GCM` | Symmetric encryption for documents and DEK wrapping |
| `SRP (Secure Remote Password)` | Password-authenticated key exchange — server never sees password |
| `HSM / KMS` | Sealing wrapped DEKs at rest on the server |

---

## Security Properties

- **Zero-knowledge server** — server holds no passwords, no plaintext, no unwrapped keys
- **Per-document DEKs** — compromise of one document key does not compromise others
- **Breach resilience** — a full database dump yields only ciphertext with no decryption path
- **Authenticated encryption** — GCM auth tags ensure ciphertext integrity and detect tampering
- **No plaintext persistence** — decrypted content lives in memory only, never written back

---

## Project Structure (Planned)

```
onekey/
├── app/                    # Next.js App Router
│   ├── (auth)/             # Login, register, SRP flows
│   ├── (dashboard)/        # Password vault, document manager
│   └── api/                # API route handlers
├── lib/
│   ├── crypto/             # Client-side crypto primitives
│   ├── db/                 # PostgreSQL client + RLS policies
│   └── aws/                # S3, KMS integrations
├── components/             # UI components
├── infra/                  # Docker, CI/CD configs
└── docs/                   # Architecture diagrams, specs
```

---

## Roadmap

- [ ] Finalize cryptographic specification
- [ ] Set up Next.js project with TypeScript
- [ ] Implement SRP authentication flow
- [ ] Build client-side crypto module (key derivation, AES-256-GCM)
- [ ] PostgreSQL schema with Row-Level Security policies
- [ ] Password vault CRUD (encrypted)
- [ ] Document upload / download flows
- [ ] AWS infrastructure setup (EC2, RDS, S3, KMS)
- [ ] Containerized CI/CD via GitHub Actions
- [ ] Audit logging
- [ ] Security audit & penetration testing

---

## Contributing

This project is in early architecture phase. Implementation has not started. Contributions, feedback on the cryptographic design, and security reviews are welcome.

---

## License

MIT — see [LICENSE](./LICENSE) for details.

---

> *"The server never sees plaintext" is not a feature. It is a constraint baked into the architecture.*