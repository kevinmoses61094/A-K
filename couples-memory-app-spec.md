# Us — A Private Calendar for Two

**Product spec & technical architecture**

---

## 1. Product thesis

The app is not a social network. It is a **shared calendar that happens to hold posts**. Every other feature (feed, comments, reactions) is secondary to the core loop:

> Open app → see hearts on meaningful dates → tap a date → relive the memory.

Design and engineering priorities, in order: **privacy → simplicity → reliability → emotional experience → performance → extra features.**

---

## 2. Recommended tech stack

| Layer | Choice | Why |
|---|---|---|
| Client | **React Native + Expo** | Single codebase → iOS + Google Play from one submission pipeline. Expo's managed workflow handles push notifications, image picker, and OTA updates without native build maintenance overhead — important for a two-person MVP team. |
| Backend | **Supabase** | Postgres (relational fit for Users/Couples/Posts/Media/Comments/Reactions), built-in **Row Level Security** (the single most important feature for "couple A can never see couple B's data"), Storage buckets with per-row access policies, Realtime channel for live sync, and Auth (email/password + OTP) out of the box. |
| Media storage | **Supabase Storage** (S3-compatible), private buckets, signed URLs | No public URLs ever exist for uploaded media — satisfies the "no public URL exposure" requirement natively. |
| Push notifications | **Expo Notifications** + Supabase Edge Function trigger on insert | Triggered server-side on new post/comment/reaction, not client-side, so it can't be spoofed. |
| State/sync | Supabase Realtime (Postgres change feed) subscribed per `couple_id` | Both partners' calendars update live without polling. |

**Why not Flutter/Firebase:** Flutter is equally valid; React Native + Expo is preferred here because Supabase's Postgres + RLS model maps directly onto the relational schema already sketched in the brief, and gives free, fine-grained "couple-only" data isolation without hand-rolled Firestore security rules, which are easier to get subtly wrong for this exact multi-tenant-of-two shape.

---

## 3. Data model

```
users
  id (uuid, pk, = auth.users.id)
  name
  avatar_url
  couple_id (fk → couples.id, nullable until paired)
  created_at

couples
  id (uuid, pk)
  partner_1 (fk → users.id)
  partner_2 (fk → users.id, nullable until joined)
  invitation_code_hash        -- store a hash, never the raw code
  invitation_expires_at
  invitation_used boolean
  created_at

posts
  id (uuid, pk)
  couple_id (fk → couples.id)
  user_id (fk → users.id)      -- author
  memory_date (date)           -- the calendar day this belongs to, independent of created_at
  title (text, nullable)
  caption (text, nullable)
  created_at
  updated_at
  deleted_at (nullable)        -- soft delete for "recently deleted"

media
  id (uuid, pk)
  post_id (fk → posts.id)
  media_type ('photo' | 'video')
  storage_path                 -- private bucket path, never public
  thumbnail_path
  width, height, duration_seconds (nullable)
  order_index                  -- position within the post

comments
  id (uuid, pk)
  post_id (fk → posts.id)
  user_id (fk → users.id)
  text
  created_at

reactions
  id (uuid, pk)
  post_id (fk → posts.id)
  user_id (fk → users.id)
  reaction_type ('heart' | ...)
  created_at
  UNIQUE (post_id, user_id, reaction_type)   -- one reaction of a kind per person
```

**Calendar ↔ posts relationship:** the calendar view runs one aggregate query per visible month —
`SELECT memory_date, COUNT(*) FROM posts WHERE couple_id = ? AND deleted_at IS NULL GROUP BY memory_date`
— giving every date its heart + count in a single round trip. Tapping a date fetches that date's posts (with media/comments/reactions joined) only when opened, so the month view stays light.

---

## 4. Authentication & invitation-code system

**User 1 (creator):**
1. Signs up (Supabase Auth, email/password or magic link).
2. Creates a `couples` row with `partner_1 = self`.
3. Server (Edge Function) generates a cryptographically random 8–10 character code (e.g. base32, excludes ambiguous characters like 0/O, 1/I).
4. Only the **hash** (SHA-256) of the code is stored in `invitation_code_hash`; the plaintext code is returned once to the client to share, then discarded server-side.
5. Code carries a short expiry (e.g. 72 hours) and `invitation_used = false`.

**User 2 (joiner):**
1. Signs up.
2. Enters the code → client sends it to an Edge Function (never a direct table write) → function hashes the input and compares.
3. On match **and** `invitation_used = false` **and** not expired: sets `partner_2 = self`, flips `invitation_used = true`, invalidates the code permanently.
4. Any other party who discovers the code gets nothing but a "join" attempt — they still need an account, and even a correct guess only ever grants access to that one empty couple space, not to any content, since content queries are always scoped by `couple_id` via RLS.

**Row Level Security (the actual privacy mechanism, not the app-store obscurity):**
```sql
-- Example policy shape, applied to posts/media/comments/reactions
create policy "couple members only"
on posts for all
using (
  couple_id in (
    select couple_id from users where id = auth.uid()
  )
);
```
Every table with a `couple_id` (directly or via `post_id → couple_id`) gets an equivalent policy. This is enforced at the database layer, so even a compromised or buggy client can't cross couple boundaries.

---

## 5. Screens

| Screen | Purpose | Key elements |
|---|---|---|
| **Onboarding — Welcome** | Choose "Create a space" or "Join partner" | Two large actions, minimal copy |
| **Sign up / Log in** | Auth | Email/password, magic link |
| **Create Couple Space** | Generates + displays invite code | Big code display, "Share" (native share sheet), "Copy" |
| **Join Partner** | Enter code | Single code input, inline validation |
| **Calendar (home)** | Primary navigation | Month grid, swipe between months, today highlighted, heart + count per date, floating "Add Memory" button |
| **Daily Memory View** | All posts for a date | Vertically scrollable post cards: media carousel, caption, author avatar + name, timestamp, react/comment row |
| **Add Memory** | Create a post | Date picker (defaults to tapped date), media picker (multi-select), caption field, optional title, preview step, Publish |
| **Edit Memory** | Modify existing post | Same form, pre-filled, delete option with confirmation |
| **Our Memories** | Reverse-chronological feed of all posts | Same post card component as Daily view |
| **Post detail / full-screen media** | Full-screen photo viewer, video playback | Swipe between media in a post |
| **Settings** | Account + couple + privacy | Partner info, notification toggles, biometric lock (future), export, delete account, recently deleted |

**Navigation:** bottom tab bar — `Calendar · Add Memory · Our Memories · Settings`, exactly as specified. "Add Memory" opens a modal stack rather than a persistent tab content area.

---

## 6. Media upload flow

1. User selects photos/videos from the device library (`expo-image-picker`).
2. Client compresses images client-side (e.g. resize to a max dimension, re-encode) before upload; large videos are transcoded/compressed where feasible.
3. Client uploads directly to a **private** Supabase Storage bucket at a path scoped by couple: `couple_id/post_id/filename`, with a bucket policy mirroring the RLS pattern above.
4. A lightweight thumbnail is generated (client-side for images; a server-side Edge Function or on-upload trigger for video posters) and stored alongside.
5. Upload progress is shown per file; failures are retried with exponential backoff and marked clearly if they ultimately fail, without blocking the rest of the post.
6. Media is only ever served via short-lived **signed URLs** requested at render time — nothing is ever public.

---

## 7. MVP scope (build in this order)

1. Auth (sign up/login)
2. Create couple space + invite code generation
3. Join couple via code
4. Calendar month view with heart/count aggregation
5. Add Memory (photos/video/caption, multi-file upload)
6. Daily Memory View (multiple posts per date, chronological)
7. Edit/delete post with confirm + soft-delete
8. Realtime sync between partners
9. Reactions + comments (lightweight)
10. Push notifications (new post, new comment)
11. Settings (partner info, notification toggle, account deletion)

Everything under "Future Features" in the brief (on-this-day, milestones, maps, chat, search, widgets, biometric lock, photo-book export) is intentionally deferred — the schema above already supports most of them without migration (e.g. tags/categories would just be a new join table; locations would be nullable lat/lng columns on `posts`).

---

## 8. Risks & App Store / Play Store considerations

- **App Store review — private content:** Apple reviewers need a way in. Provide a demo account with a pre-paired demo couple space in the review notes, since reviewers cannot generate a second device to test the invite flow themselves.
- **Account deletion requirement:** both stores require in-app account deletion (not just deactivation) — build this into Settings from day one, including cascading deletion of media from Storage.
- **Data collection disclosure:** App Store "Privacy Nutrition Label" and Play "Data safety" form must accurately list photos/videos/identifiers collected — get this right before first submission to avoid rejection loops.
- **Push notification permission timing:** ask for notification permission contextually (e.g. after the first successful pairing), not on first launch, to avoid a wall of iOS permission prompts and low opt-in.
- **Large media uploads on cellular:** warn or default to Wi-Fi-only upload for video to avoid failed uploads / data-cost complaints.
- **Invitation code abuse:** rate-limit join attempts per account/IP in the Edge Function to prevent brute-forcing an 8–10 character code.
- **Orphaned couple spaces:** handle the case where `partner_2` never joins (expired invite) — allow regenerating a new code without creating a duplicate `couples` row.
