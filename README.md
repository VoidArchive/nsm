# 🗺️ Nepal Shame Map (KSM)

A civic tech project built to expose Nepal’s environmental neglect—**in real time, by real people**.

This is not a feel-good map. It is a mirror held up to a broken system.

---

## ❓ What Is KSM?

The **Nepal Shame Map** is a public, crowdsourced platform that lets citizens:

- 📍 Report pollution incidents (trash, plastic fires, sewage leaks, air pollution)
- 🗺️ Pin them directly on a live map of Nepal
- 📷 Upload photos as evidence
- 📊 Make environmental shame **visible, undeniable, and trackable**

Think of it as the **Google Maps of civic failure**—only every marker is a call to action.

---

## 🚀 Tech Stack

| Layer            | Tech                                 |
| ---------------- | ------------------------------------ |
| Frontend         | [SvelteKit](https://kit.svelte.dev)  |
| Map Rendering    | [Leaflet.js](https://leafletjs.com)  |
| Auth & Database  | [Supabase](https://supabase.com)     |
| Storage (Images) | Supabase Storage                     |
| Testing (E2E)    | [Playwright](https://playwright.dev) |
| Deployment       | Cloudflare / Vercel (TBD)            |

---

## 🔐 Security & Integrity

- ✅ **RLS (Row-Level Security)** enforced in Supabase to protect user-submitted data
- ✅ Insert policies ensure users can only write their own data
- ✅ Anonymous or email-based auth for minimal barrier to contribution

---

## 📦 Getting Started (Local Dev)

> Requires: `pnpm`, `Node.js`, `Supabase CLI`, and Supabase project initialized

```bash
# Clone the repo
git clone https://github.com/VoidArchive/nsm.git
cd nsm

# Install deps
pnpm install

# Start dev server
pnpm dev
```

Set up your .env based on the Supabase keys.

## 📌 Reporting Flow

    1. User clicks on the map to select location
    2. Fills in pollution type, optional description, and photo
    3. Submits the form
    4. Report is stored in Supabase and rendered back on the map

## 🧠 Vision

This is Phase 1 of a larger initiative to

- Mobilize public shame into environmental pressure
- Create a public record of environmental abuse
- Push the municipality into accountability—by making neglect unignorable

🤝 Contribute
If you’ve got skills—frontend, backend, design, activism—you’re welcome. But don’t come to play safe.

## 📍 Status

✅ Base map rendering using Leaflet

✅ Pollution report submission

✅ Image upload to Supabase

✅ Basic RLS security

🟡 Aggregation & filters (coming soon)

🟡 Public sharing / social callouts (planned)
