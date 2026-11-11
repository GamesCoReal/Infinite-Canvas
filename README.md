# Infinite Canvas

A browser-based infinite drawing canvas. Cloud sync uses Google sign-in through Supabase; each account's canvases are private.

## Run it

1. Configure Google sign-in and the Supabase table below.
2. Serve this folder over HTTP. For example, from this folder run `py -m http.server 8000`, then open `http://localhost:8000` in a modern browser.
3. Continue with Google. Existing canvases in this browser are copied to the first signed-in account; canvases from that account then sync between devices.

The app also keeps a local IndexedDB copy for fast loading and migration. Browser private mode, clearing site data, or uninstalling/resetting the browser can erase that local copy. The Supabase account copy remains available after signing back in.

## Supabase setup

The browser app is configured with the project URL and public publishable key. These are intended for client-side use; never put a `service_role`, OAuth client secret, or other secret in this file.

In the Google Cloud Console, create an OAuth client ID for a **Web application**. Add the Supabase callback URL (`https://<project-ref>.supabase.co/auth/v1/callback`) to its authorized redirect URIs. In Supabase, open **Authentication → Sign In / Providers → Google**, enable Google, and enter the Google client ID and client secret there.

In **Authentication → URL Configuration**, set the Site URL to the address used to open the app and add it to the Redirect URLs allow list (for the example above, `http://localhost:8000/**`). Configure any deployed app URL there as well. Run this SQL once in **SQL Editor**:

```sql
create table if not exists public.canvases (
  id text not null,
  owner_id uuid not null references auth.users(id) on delete cascade,
  name text not null,
  payload jsonb not null,
  share_code text,
  updated_at timestamptz not null default now(),
  primary key (owner_id, id)
);

alter table public.canvases enable row level security;

grant select, insert, update, delete on public.canvases to authenticated;
grant usage on schema public to anon, authenticated;

drop policy if exists "Users can read their own canvases" on public.canvases;
create policy "Users can read their own canvases"
  on public.canvases for select to authenticated
  using ((select auth.uid()) = owner_id);

drop policy if exists "Users can create their own canvases" on public.canvases;
create policy "Users can create their own canvases"
  on public.canvases for insert to authenticated
  with check ((select auth.uid()) = owner_id);

drop policy if exists "Users can update their own canvases" on public.canvases;
create policy "Users can update their own canvases"
  on public.canvases for update to authenticated
  using ((select auth.uid()) = owner_id)
  with check ((select auth.uid()) = owner_id);

drop policy if exists "Users can delete their own canvases" on public.canvases;
create policy "Users can delete their own canvases"
  on public.canvases for delete to authenticated
  using ((select auth.uid()) = owner_id);

create unique index if not exists canvases_share_code_unique
  on public.canvases (share_code)
  where share_code is not null;

create or replace function public.get_shared_canvas(p_share_code text)
returns table (id text, name text, payload jsonb)
language sql
stable
security definer
set search_path = ''
as $$
  select c.id, c.name, c.payload
  from public.canvases as c
  where c.share_code = p_share_code
    and p_share_code ~ '^[A-Fa-f0-9]{10,64}$'
  limit 1;
$$;

create or replace function public.update_shared_canvas(
  p_share_code text,
  p_payload jsonb
)
returns boolean
language plpgsql
security definer
set search_path = ''
as $$
begin
  if pg_catalog.jsonb_typeof(p_payload) <> 'object'
     or p_share_code !~ '^[A-Fa-f0-9]{10,64}$' then
    raise exception 'Invalid shared canvas request';
  end if;

  update public.canvases
  set payload = p_payload, updated_at = pg_catalog.now()
  where share_code = p_share_code;

  return found;
end;
$$;

revoke all on function public.get_shared_canvas(text) from public;
revoke all on function public.update_shared_canvas(text, jsonb) from public;
grant execute on function public.get_shared_canvas(text) to anon, authenticated;
grant execute on function public.update_shared_canvas(text, jsonb) to anon, authenticated;
```

Run this complete SQL in **SQL Editor**, including the RPC functions, to enable room links. If the private `canvases` table is already set up, you can rerun the script: it replaces the owner policies and adds the room RPC functions. Table row-level security keeps the library private to its owner. The RPC functions expose only the canvas matching a high-entropy room code and allow editing that canvas's drawing payload. Anyone who obtains a room link can view and edit that room; treat the URL like a password. Remove its `share_code` from the row to revoke the link.

## Controls

- Canvases: sign in, then enter your display name if prompted. Open Canvases to create a canvas, switch canvases, or view square previews. The empty library has a plus tile that creates a canvas, opens it, and asks for its name. Press and hold a canvas tile to select it; select multiple tiles to delete them, or select one to rename it.
- Accounts: use **Continue with Google**. Sign out from the canvas library. On the first sign-in, canvases already stored in this browser are migrated to that Google account.
- Share: create a link with an automatic room code. People with the link can open and edit that canvas without access to the owner's other canvases. Use a public HTTPS-hosted app URL; localhost and file URLs cannot be opened by other people.
- Draw: drag with mouse, finger, or stylus.
- Paint tools: click the bottom-center tool circle to reveal the fixed tool row. Select a drawing tool, then click it again to open its settings. Pen, Marker, Highlighter, Calligraphy, and Flat Pencil keep their own size, color, and fill settings; Highlighter also has a transparency slider. Shader smudges colors from the canvas. New strokes keep their on-screen size as you zoom; older saved strokes keep their original zoom behavior.
- Drawing from open settings: click the canvas to close settings without drawing, or drag to draw and close settings.
- Lasso: circle a region and reconnect to select its stroke portions. Drag the selection to move it, or drag its top handle to rotate it.
- Ruler: select Ruler to show it in the middle of the screen. Drag its surface to move it; right-click-drag or use two fingers on it to move and rotate it. Drawing pauses where the ruler covers the canvas.
- Zoom and saved views: use the mouse wheel or pinch to zoom from 0.0000001% to 100,000,000,000%. The wheel zooms around the pointer, and the canvas can be panned without boundaries. The canvas starts centered at 50% each time it opens. Use Saved locations at the top-left to name and revisit a view's center and zoom.
- Nib angle: Highlighter, Calligraphy, and Flat Pencil have a rotation slider and reset button in their settings. Right-click and drag on the canvas to adjust the selected tool's nib angle.
- Brush cursor: the outline shows the current brush size. The eraser outline switches between black and white for contrast.
- Dark mode: use the top-right button to switch the canvas between light and dark; black and white canvas colors swap, and the choice is autosaved.
- Lasso outline: dotted marks switch between white and black based on the canvas beneath them.
- Fill: when enabled, closing a stroke fills its enclosed area; leaving it open fills the connected area across the infinite board.
- Pan: hold Space and drag; on touch, use two fingers.
- Undo / redo: always-visible arrows at the bottom-right (redo is disabled until an undo), or Ctrl/Cmd+Z and Ctrl/Cmd+Y.

## How it works

- Strokes are stored as vector point lists in a master `Map`.
- A spatial chunk index maps each 1024 × 1024 world-space chunk to stroke IDs.
- New strokes keep their width in screen pixels and are indexed by their centerlines; saved older strokes retain world-space widths.
- Rendering checks chunks intersecting the viewport and a margin for visible brush widths.
- A stroke crossing chunk boundaries is indexed in every chunk it touches, but its geometry is stored only once.
- Very large strokes are kept outside the chunk index so drawing remains practical at extreme zoom levels.
- The canvas autosaves to IndexedDB after edits and camera movement, then syncs to the signed-in Supabase account.
- Saved locations, including their view center and zoom level, are stored in the same local IndexedDB document.

## Storage and privacy

IndexedDB is a local cache; Supabase is the signed-in account's cloud copy. The app uses the Supabase JavaScript client from jsDelivr, so an internet connection is required for account sign-in and cloud sync. Room-code links do not bypass the owner-only row-level security policies.
