# CBS doc comments — Supabase setup

The email-copy doc (`CBS_Nurture_Email_Copy.html`) now has a highlight-to-comment
layer. Anyone with the link can select text and leave a comment; everyone sees the
same comments. To make comments shared (not just saved in one browser), do the
one-time Supabase setup below. Takes about 5 minutes.

Until you add the keys, the doc still works — comments just save to each person's
own browser, and a maroon banner reminds you it isn't shared yet.

## 1. Create a free Supabase project

1. Go to https://supabase.com and sign up (free tier is plenty).
2. Click **New project**. Name it anything (e.g. `vinyl-cbs-comments`). Pick a region near you. Set a database password (you won't need it again for this).
3. Wait ~1 minute for it to provision.

## 2. Create the comments table

In the project, open **SQL Editor** (left sidebar) → **New query**, paste this, and click **Run**:

```sql
create table public.comments (
  id         uuid primary key default gen_random_uuid(),
  doc_id     text not null,
  thread_id  uuid,              -- null = top-level comment; set = reply to that comment
  author     text,
  body       text not null,
  quote      text,              -- the highlighted text
  prefix     text,              -- ~40 chars before (for re-anchoring)
  suffix     text,              -- ~40 chars after
  start_off  int,
  end_off    int,
  resolved   boolean not null default false,
  created_at timestamptz not null default now()
);

create index comments_doc_idx on public.comments (doc_id, created_at);

-- Anyone with the link can read/add/update (resolve). No deletes.
alter table public.comments enable row level security;

create policy "read"   on public.comments for select using (true);
create policy "insert" on public.comments for insert with check (true);
create policy "update" on public.comments for update using (true) with check (true);
```

> Note on access: these policies let anyone with your anon key read and post
> comments. That matches "anyone with the link can comment." The anon key is a
> public key (safe to ship in the HTML) — it is not your secret service key.
> Don't paste the `service_role` key anywhere.

## 3. Enable realtime (optional but nice)

**Database → Replication** (or **Realtime**) → enable replication for the
`comments` table. This makes new comments appear for others without a refresh.
If you skip it, the doc still polls every 30 seconds.

## 4. Get your two keys

**Project Settings → API**. Copy:

- **Project URL** (e.g. `https://abcd1234.supabase.co`)
- **anon public** key (the long one labeled `anon` / `public`)

## 5. Paste them into the HTML

Open `CBS_Nurture_Email_Copy.html`, find the `CBS_COMMENTS_CONFIG` block near the
bottom, and replace the placeholders:

```js
const CBS_COMMENTS_CONFIG = {
  SUPABASE_URL:      "https://abcd1234.supabase.co",
  SUPABASE_ANON_KEY: "eyJhbGciOi...your anon key...",
  DOC_ID:            "cbs-nurture-email-copy"
};
```

Commit, deploy to Vercel, and the maroon banner disappears. Comments are now shared.

## Reusing on the Strategy page

To add comments to `CBS_Nurture_Sequence_Strategy.html` too:

1. Add `data-comment-root` to its main content wrapper (the `<div class="container">`).
2. Copy the entire commenting block (the `<style>`, the `<div id="cbs-cmt-app">`, and the two `<script>` tags) from the bottom of the email-copy file to just before `</body>`.
3. Change `DOC_ID` to something unique, e.g. `"cbs-nurture-strategy"`, so the two docs keep separate comment threads in the same table.

## How anchoring survives edits

Each comment stores the highlighted text plus ~40 characters of surrounding
context. If the copy is edited, the script re-finds the best matching spot. If the
exact text is deleted, the comment still shows in the sidebar marked
"(text changed - unanchored)" so nothing is silently lost.
