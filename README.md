# Casa Prefab

> A tiny self-hosted project tracker, built end-to-end with AI. One HTML file, your own Supabase, realtime sync.

**Live:** [luisgrandegg.github.io/project-tracker](https://luisgrandegg.github.io/project-tracker/)

A small project tracker designed to fit on a single page of HTML. It uses your own Supabase project as a backend, syncs in real time across devices, and ships as one file you can open in a browser, drop on a static host, or send to a collaborator.

Built originally to coordinate a small house build — an architect, a foundations contractor, a prefab manufacturer, and the two owners trying to keep everyone aligned. General enough for any small project where two or three people need to share a list of who-owes-what-by-when.

## Features

- **List view** — items with status, owner, due date, labels, and a per-item activity log. Sortable by status, owner, due date, title, created date, or a smart default (overdue first, done last).
- **Dashboard view** — distributions of items by status, owner, deadline, and label. Each bar drills into a filtered list.
- **Timeline view** — an Expected vs Real cumulative chart (planned due dates vs actual completions) and a Gantt with overdue extensions and completion ticks.
- **Labels** — free-form tags with hashed colors, autocomplete from existing labels, and case-insensitive dedupe. Click any chip on a card to filter.
- **Activity log per item** — emails, WhatsApp messages, calls, meetings, notes; date-stamped entries that travel with each item.
- **Realtime sync** — when one person adds, edits, or deletes something, others see it within a couple of seconds. Powered by Supabase Postgres realtime.
- **Persisted preferences** — view, sort, and active label filter all survive reloads, separately on each device.
- **No build step. No framework. No backend code of your own.** One HTML file pointed at a Supabase project.

## Setup

### 1. Create a Supabase project

Sign up at [supabase.com](https://supabase.com) and create a project. The free tier is plenty. Pick any name.

### 2. Run the setup SQL

In your project's SQL Editor, paste and run this:

```sql
create table public.tracker_items (
  id uuid primary key default gen_random_uuid(),
  title text not null,
  description text default '',
  status text not null,
  owner text not null,
  due_date date,
  start_date date,
  completed_at timestamptz,
  created_at timestamptz default now(),
  updated_at timestamptz default now(),
  log jsonb default '[]'::jsonb,
  labels text[] default '{}'
);

alter table public.tracker_items enable row level security;
create policy "allow_anon_all" on public.tracker_items
  for all to anon, authenticated using (true) with check (true);

alter publication supabase_realtime add table public.tracker_items;
```

This creates the table, opens it to anon access, and enables realtime updates.

### 3. Host the HTML file

Serve `project-tracker.html` from any static host:

- **Cloudflare Pages** or **Netlify** — drag-and-drop deployment, free, ~2 minutes
- **GitHub Pages** — free, automatic HTTPS, serves the file directly from this repo. This repo deploys via `.github/workflows/pages.yml` to [luisgrandegg.github.io/project-tracker](https://luisgrandegg.github.io/project-tracker/)
- **Locally** — double-click the file to open it in your browser

Hosting remotely is the right move if more than one person needs access.

### 4. Configure on first load

When you open the page, you'll see a setup screen asking for your Supabase Project URL and the publishable / anon key (find them in **Project Settings → API**). Paste them, click Configure, and the credentials are saved to localStorage on that browser. Next time you open the page, it goes straight to the tracker.

A second person joining: they open the same page, paste the same URL and key once, and they see the same tracker — with changes from either side syncing live.

## Customization

The list of owners and statuses is hardcoded in the JS — search for `OWNERS`, `OWNER_DISPLAY`, and `STATUSES` at the top of the script and edit. No migration needed unless you change values that already exist in the database.

The visual aesthetic is driven by a CSS variables block at the top of `<style>`. Change `--bg`, `--ink`, `--accent`, and the status `--s-*-bg/fg` variables to repaint everything.

## How it works

One static HTML file with inline CSS and JS. No bundler, no `package.json`, no `node_modules`. The JavaScript uses [supabase-js v2](https://github.com/supabase/supabase-js) loaded from a CDN to talk to your Supabase project's PostgREST and realtime endpoints directly.

State lives in a single `state` object. Render functions read from it and write innerHTML strings; click handlers attach via `data-action` attributes after each render. No virtual DOM, no reactivity framework. When state changes, the relevant render function is called.

The script is organized into clearly-marked sections: config, constants, state, helpers, Supabase layer, render layer, modal, view switching, setup wiring, init. See `CLAUDE.md` for more architectural detail.

## What this isn't

- **Multi-tenant.** One Supabase project per tracker. Don't share the URL and key with people you wouldn't trust to read and write everything.
- **Authenticated.** The RLS policy allows anon access. The "secret" is the obscurity of your project URL and publishable key. For a small private team this is acceptable; for anything sensitive, add Supabase Auth and tighten the policy.
- **Scaled.** Tested at small numbers of items (dozens, not thousands). The list and dashboard re-render in full when state changes — no virtualization.
- **A template.** It's a personal tool published openly. If it fits, fork it; if it almost fits, fork it and edit; if it doesn't fit, leave it.

## Built with AI

This was built end-to-end with [Claude](https://claude.ai), iterating in conversation from "I want to build a tracker for the project" to the finished single-file app you see here — including this README. The `CLAUDE.md` in the repo describes the conventions and constraints to anyone (human or AI) extending it.

## License

MIT. Do whatever you want with it.
