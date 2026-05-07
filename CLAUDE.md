Casa Prefab — Project Tracker
A single-file HTML tracker for managing a small construction project. Built for two
end users (Luis and his partner Mathi) coordinating a Spanish prefabricated house
build across multiple contractors and an architect. Personal tool — sized for two
people, not a product.
Stack

Frontend: one static HTML file. Vanilla JS, no framework, no build step. Fonts
via Google Fonts, supabase-js@2 via CDN.
Backend: a Supabase project (Postgres + PostgREST + realtime). The HTML talks to
it directly using the publishable / anon key.
Hosting: served as a static file (Cloudflare Pages, Netlify, GitHub Pages, or
opened locally in Chrome).

Files

project-tracker.html — the entire app. CSS, HTML, and JS all inline.

There is no package.json, no node_modules, no bundler. Opening the file in a
browser is "running" it.
Critical constraints
These exist because the project is sized for two people and meant to stay maintainable
by one person without ongoing engineering work. Don't violate them without an explicit
ask.

Single-file deployment. Don't split into multiple HTML / CSS / JS files. Don't
add a build step. The whole app must remain copyable as one file.
Vanilla JS only. No React, Vue, Svelte, etc. No bundler. No TypeScript. The CDN
load of supabase-js is the only external runtime dependency.
No backend code. All logic runs in the browser. Supabase provides DB +
realtime. If you'd reach for a custom API endpoint, prefer a Supabase view, an
RPC, or client-side computation.
No auth (currently). The RLS policy allows anon access. The "secret" is the
obscurity of the project URL + publishable key, which has been accepted as
acceptable for a 2-person tool. Don't add auth unless asked. If you do add it,
tighten the RLS policy in the same change.
Mobile must work. Both users access this from phones occasionally. The existing
responsive breakpoint is ~720px; preserve it when adding new views.

Architecture
One inline <script> block, organized in clearly-marked sections. Read in this order:

Supabase config — SUPABASE_URL and SUPABASE_KEY are let-bound mutables
loaded from localStorage. The setup flow updates them at runtime.
Constants — STATUSES, OWNERS, LABEL_COLORS, OWNER_COLORS, sort/view
keys.
State — single state object. No external state library. Render functions
read directly.
Helpers — uid, escapeHtml, date formatting, getTimeStatus.
Supabase layer — rowToItem / itemToInsertRow mapping, fetch / create /
update / delete, testConnection, subscribeRealtime.
Render layer — renderTracker dispatches to renderItems / renderDashboard
/ renderTimeline based on state.view. Cards built as innerHTML strings (no
virtual DOM). Click handlers wired via data-action attributes after setting
innerHTML.
Modal — openModal / closeModal / saveModal. Labels held in
state.modalLabels because the chip input mutates across re-renders.
View switching — setView toggles which top-level container is visible.
Persists to localStorage.
Setup wiring — wireSetupView, handleConfigureClick, reconfigure.
Init — connect() is the entry point. If credentials are missing, routes to
the setup view; otherwise connects, fetches, renders, subscribes.

Database schema
Current shape of public.tracker_items:
ColumnTypeNotesiduuid PKdefault gen_random_uuid()titletext NOT NULLdescriptiontextdefault ''statustextone of: To plan / To do / In progress / Action Required / Doneownertextone of: Damaso / Luis y Mathi / Colven / Isidorodue_datedatenullablestart_datedatenullable; used by the Ganttcompleted_attimestamptznullable; auto-managed by the client when status flips to / from Donecreated_attimestamptzdefault now()updated_attimestamptzclient sets on each updatelogjsonbarray of {id, type, date, content}labelstext[]default '{}'attachmentsjsonbdefault '[]'; array of {id, name, type, size, data, addedAt} where data is a base64 data URL. Images are stored inline in the row (favoring DB encoding over file hosting). Client downscales to 1600px longest edge and re-encodes to JPEG before saving.
RLS is enabled with a single policy allow_anon_all granting full access to anon
and authenticated roles. Realtime is enabled via
alter publication supabase_realtime.
When schema changes are needed, hand the user a SQL migration snippet to run in their
Supabase SQL Editor and update the table above. Don't introduce a migrations
directory — there's no infrastructure for it.
Conventions

Status sort order is by urgency, not alphabetical. Action Required first,
Done last. See STATUS_ORDER — used in the dashboard chart and the "Sort by
status" option.
Owner display vs storage. Stored as ASCII keys (Damaso), displayed via the
OWNER_DISPLAY map (Dámaso). Don't change stored values without a migration.
Label dedupe is case-insensitive. "Urgent" and "urgent" collapse to one label,
keeping whichever casing was used first. See getAllLabels.
Label colors are hashed, not picked. Same label name always renders the same
color via labelStyle(). The palette is a fixed muted earth-tone set in
LABEL_COLORS.
Time calculations live in getTimeStatus(). Returns
{ type, label, className, days } where type is one of
done | none | today | future | overdue. Used by the list card pills and the
dashboard's deadline distribution.
Dates are stored as YYYY-MM-DD strings (no time component). Use dayDate()
to parse — new Date('YYYY-MM-DD') parses as UTC, which causes timezone drift on
cumulative comparisons.
Realtime updates skip re-rendering while editing. subscribeRealtime's
callback checks state.editingId and state.addingLogTo and bails so it doesn't
yank content from under the user mid-keystroke. The fetch still runs; only the
visual update is deferred.
completedAt is auto-managed. Set to now() when status transitions to
Done, cleared when it transitions out. Logic lives in
handleItemAction('mark-done') and saveModal. Don't expect the user to set it
manually.

Aesthetic
Editorial / Mediterranean — warm cream backgrounds, terracotta accent, muted
earth-tone status colors. Three typefaces:

Fraunces (variable serif) for display: titles, chart titles, item names
DM Sans for body text and buttons
JetBrains Mono for technical labels: dates, counts, status pills

CSS variables for the palette are defined at the top of <style> (--bg, --ink,
--accent, status --s-*-bg/fg, etc.). Use them — don't hardcode hex values.
localStorage keys
Browser-scoped, per-device:
KeyPurposetracker_supabase_urlProject URL set during setuptracker_supabase_keyPublishable / anon key set during setuptracker_view_v1Last selected view: list / dashboard / timelinetracker_sort_v1Sort preference, JSON {by, dir}tracker_filter_label_v1Active label filter
project_tracker_v1 is a legacy key from an earlier prototype that used the Claude
artifact runtime's window.storage. There's still migration code in
migrateLegacyIfPresent that imports any items found under that key on first
connect, then clears it. Don't reuse the key for anything new.
Testing changes
There are no automated tests. After a non-trivial change:

Open the file in Chrome with the DevTools console open. No errors should appear.
Verify the basic flows: switch views, open and save an item, add a log entry, mark
something Done.
Open a second window pointed at the same Supabase project and verify changes from
one window propagate to the other within a couple of seconds (realtime).
Resize to ~380px wide and check the layout stays readable.

Things to push back on
If asked, raise concerns and confirm before doing any of these:

Adopting a frontend framework
Splitting into multiple files
Adding a backend service
Adding a test suite (overkill for a personal tool)
Storing sensitive data in localStorage beyond the existing Supabase credentials

Common changes the user makes

Adding new fields to items — always involves a SQL migration handed back to the
user
New views — a renderXxx function, a tab in view-tabs, a case in setView,
persistence already handled via tracker_view_v1
New filters or sort dimensions — extend compareItems and the toolbar
Visual tweaks within the existing aesthetic
