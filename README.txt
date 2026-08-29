BABE & PAPA — SETUP

1) SUPABASE TABLES/BUCKETS NEEDED

   -- submissions (her uploads for you to approve/deny)
   create table submissions (
     id uuid primary key default gen_random_uuid(),
     file_url text not null,
     file_type text not null,
     status text not null default 'pending',
     created_at timestamptz not null default now()
   );
   alter table submissions enable row level security;
   create policy "anyone can insert" on submissions for insert to anon with check (true);
   create policy "anyone can read" on submissions for select to anon using (true);
   create policy "anyone can update status" on submissions for update to anon using (true);

   Bucket: comic-uploads (public, anon insert+select policy)

   -- comic_pages (the actual comic, managed from admin)
   create table comic_pages (
     id uuid primary key default gen_random_uuid(),
     page_number int not null,
     image_url text not null,
     created_at timestamptz not null default now()
   );
   alter table comic_pages enable row level security;
   create policy "anyone can read" on comic_pages for select to anon using (true);
   create policy "anyone can insert" on comic_pages for insert to anon with check (true);
   create policy "anyone can update" on comic_pages for update to anon using (true);
   create policy "anyone can delete" on comic_pages for delete to anon using (true);

   Bucket: comic-pages (public, anon insert+select+delete policy)

2) HOW IT WORKS FOR HER
   - Opens index.html → "Make me happy — upload something you like"
   - Uploads a photo/video → sees "Sent! Waiting for approval..." (auto-checks every few seconds)
   - You approve → her page unlocks straight into the comic, and stays unlocked
   - You deny → she sees "Not this time — try again?" and can upload again

3) HOW IT WORKS FOR YOU (ADMIN)
   - Tap the small 🔒 icon, bottom-right corner
   - Enter PIN: 251013
   - Pending/Approved/Denied/All tabs to review her uploads
   - "📖 Pages" tab: add new comic pages (upload button), reorder with ↑/↓,
     or delete pages — the reader updates from whatever's here.
     No local image files needed anymore; everything lives in Supabase.

The PIN is a soft gate only — a static page can't hide it from someone
who checks "view source", so don't treat it as real security.

4) EDITABLE SITE TEXT (new)
   Run this SQL too:

   create table settings (
     key text primary key,
     value text not null
   );
   alter table settings enable row level security;
   create policy "anyone can read" on settings for select to anon using (true);
   create policy "anyone can insert" on settings for insert to anon with check (true);
   create policy "anyone can update" on settings for update to anon using (true);

   insert into settings (key, value) values
     ('gate_message', 'Make me happy — upload something you like 💌'),
     ('pending_message', 'Sent! Waiting for approval...'),
     ('denied_message', 'Not this time'),
     ('approved_message', 'Approved 💕');

   Then in admin, tap "✏️ Text" to edit the gate/pending/denied/approved
   messages any time — changes take effect the next time the page loads.

5) LOGIN (new)
   Before she sees anything, there's now a login screen:
     Username: Papa
     Password: Babeiloveyou
   Once logged in, her device remembers it (won't ask again on that
   device). Like the admin PIN, this is a soft gate only — a static
   page can't hide the password from someone checking "view source".

6) ADMIN "LOCK NOW" (new)
   In admin, there's a red "Lock Now" button in the tab bar. Tapping it
   (with confirmation) immediately revokes every approved unlock —
   her page detects this within ~5 seconds if it's open, or the next
   time she opens it, and sends her back to "upload something" with
   a "🔒 Locked — please upload again to continue." message. She has
   to upload fresh and get approved again, same as denial or the
   60s inactivity timeout.

   No extra SQL needed — this reuses the "settings" table from step 5,
   just adds a "lock_token" row automatically the first time you use it.
