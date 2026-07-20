---
name: trello
description: Access the user's Trello over the raw REST API — list boards/lists/cards, read/download attachments, post comments, upload files to cards. Use when the user mentions Trello, a board/card/list, or asks to fetch/comment/upload Trello attachments.
---

# Trello (raw-API skill)

Personal helper for the user's Trello via curl. Token-lean (responses trimmed with `fields=`), local-file upload + attachment **byte download** (the gap most MCP servers miss).

## Setup (once)
Creds live in `~/.claude/.trello.env` — three variables:
- `TRELLO_KEY`, `TRELLO_TOKEN` — get both at https://trello.com/power-ups/admin/new (see the repo README for the full walkthrough with screenshots).
- `TRELLO_MEMBER_ID` — your member ID (used by `tr_mine` / `tr_mine_board`). After `TRELLO_KEY`/`TRELLO_TOKEN` are set, run `source "$(command -v trello.sh 2>/dev/null || echo "${HOME}/.claude/skills/trello/trello.sh")" && tr_get "/1/members/me?fields=id,username"` and paste the `id` back into the env file.

Copy `.trello.env.example` from the repo to `~/.claude/.trello.env` and fill it in. If placeholders are unfilled, tell the user to fill that file. **Never print the token.**

## Usage
Source the helper, then call functions. Run via the Bash tool:
```bash
source "$(command -v trello.sh 2>/dev/null || echo "${HOME}/.claude/skills/trello/trello.sh")"
tr_me                         # verify creds -> username
tr_boards                     # boards + IDs
tr_lists  BOARD_ID            # lists on a board
tr_cards  LIST_ID             # cards in a list
tr_card   SHORTLINK [fields]  # ONE card by shortlink/ID (default: name,desc,shortUrl,idList)
tr_atts   CARD_ID             # attachments (id, name, bytes, mime)
tr_dl     CARD_ID ATT_ID FILENAME OUTPATH   # download bytes (auth header)
tr_comment CARD_ID "text"|FILE      # post comment -> returns action JSON (capture .id). Use a FILE path (Write tool, UTF-8) for anything non-trivial/non-ASCII.
tr_comment_update ACTION_ID "text"|FILE  # edit existing comment (ACTION_ID from tr_comment response)
tr_card_update CARD_ID name=FILE [desc=FILE]   # update title/description -> same FILE-path rule
tr_upload  CARD_ID FILEPATH   # attach local file to card
tr_get    "/1/..."            # raw passthrough for unwrapped endpoints
tr_mine        LIST_ID        # cards in a list assigned to me (uses TRELLO_MEMBER_ID)
tr_mine_board  BOARD_ID       # cards on a whole board assigned to me
# --- card lifecycle (create / checklist / move) — text args FILE-SAFE like tr_comment ---
tr_card_create LIST_ID NAME|FILE [DESC_FILE]   # create card -> JSON (capture .id, .shortUrl)
tr_checklist_add CARD_ID NAME                  # add checklist -> JSON (capture .id)
tr_checkitem_add CHECKLIST_ID NAME|FILE        # add item at bottom
tr_card_checklists CARD_ID                     # list checklists with id,name (use .id as CHECKLIST_ID below)
tr_checkitems CHECKLIST_ID                     # list items (id, name, state) — takes CHECKLIST_ID, not card ID
tr_checkitem_set CARD_ID ITEM_ID complete|incomplete [NAME|FILE]  # tick / untick
tr_card_move CARD_ID LIST_ID                   # advance lifecycle (In Progress->Review->Done)
```
Each Bash call is a fresh shell — `source` the helper every call.

Note: Trello has **no comment-level attachments**. "Upload at a comment" = `tr_comment` + `tr_upload` (card-level); they sit together in the card feed.

## Comment style (team-facing recaps)
Trello comments are read by all kinds of people (sales/CS/ops/devs), so write to be
globally understandable. Default to:
- **Audience = everyone.** No code/file/API names, no jargon. Say "the Claim button asks
  for the buyer's email" not "the modal POSTs to /api/.../claim".
- **Bullets, not paragraphs.** One short line per point. Lead with a plain one-line title
  (no bold — keep all text the same weight).
- **Never pass comment/title/description text as a literal shell argument.** On Windows,
  any non-ASCII char (em-dash, curly quotes, accents, emoji) passed inline gets corrupted
  before curl sees it (cp1252 byte, not UTF-8 — e.g. an em-dash silently becomes literal
  `%97` in the stored text). This isn't a "remember not to use em-dash" rule — it breaks
  silently and is easy to forget, so the fix is structural: always write the text to a
  UTF-8 file with the `Write` tool first, then pass the **file path** to `tr_comment` /
  `tr_card_update` (both detect a file path and stream its bytes via `--data-urlencode
  text@FILE`, bypassing argv entirely). Only pass literal text inline for trivial
  pure-ASCII one-liners. (macOS/Linux are UTF-8 native so inline is safer there, but the
  file form works everywhere — prefer it.) After posting, sanity-check with
  `tr_get "/1/actions/ACTION_ID?fields=data"` (comment text lives at `.data.text`,
  NOT top-level `.text` — `fields=text` returns no text key) or `tr_card CARD_ID`, and
  look for stray `%XX` sequences before moving on.
- **No decorative symbols / emoji.** Spell out units ("240 euro", not the euro sign).
- **Structure a progress recap as:**
  1. Title + one-line summary of the change.
  2. "What is new, live now:" — bullets of what shipped (user-visible terms).
  3. "<Owner> team need to work on these:" — action items grouped by who owns them,
     each a concrete next step. Put blockers here.
- Confirm the draft with the user before posting.

## Safety
- Destructive ops (delete/archive) are NOT wrapped — for those, confirm with the user, then use `tr_get` / explicit curl.
- Token = full account access. Keep it in `.trello.env` only; never echo it.

## Self-review (improvement loop)
After a multi-step op (e.g. download -> comment -> upload chain), confirm the work landed before declaring done. When something breaks or is clumsy, fix `trello.sh` and append a line to Lessons below. Persistent cross-session facts (board IDs, gotchas) -> also write to project memory.

## Lessons
- `tr_get` takes a **full API path**, NOT a bare card ID. `tr_get jF8DMKub` builds a broken
  URL (`api.trello.comjF8DMKub`) -> empty body. Either `tr_get "/1/cards/jF8DMKub?fields=name"`
  or use the wrapped `tr_card jF8DMKub`.
- Single-card lookup by shortlink: `tr_card SHORTLINK` (the 8-char code in `trello.com/c/XXXX`).
- Attachment byte download needs the `Authorization: OAuth ...` **header**; `key/token` query params alone return 401 for the file. (`tr_dl` handles it.)
- Comment permalink: `tr_comment` returns the action JSON — build the shareable URL from
  `.data.card.shortLink` + the action `.id`:
  `https://trello.com/c/<shortLink>#comment-<actionId>`.
  The short `/c/<shortLink>` path redirects to the full card; the `#comment-<id>` anchor jumps to it.
- Non-ASCII text (em-dash, curly quotes, accents) passed as a literal arg to `tr_comment`
  gets mangled on Windows (cp1252 byte -> wrong `%XX` escape, stored corrupted, e.g. an
  em-dash becomes literal `%97`). Fixed structurally — `tr_comment` and `tr_card_update`
  accept a **file path** (write the text with the `Write` tool first) and stream it via
  `--data-urlencode text@FILE`, which never touches argv. Prefer the file form.
- Image uploads auto-becoming the card cover is a **board-level** setting
  (`boards/{id}/prefs.cardCovers`), not a per-upload thing. Passing `setCover=false` on
  the `tr_upload` POST does NOT reliably stop it. `tr_upload` sends `setCover=false` as a
  harmless extra, but the actual fix (when the user wants it) is
  `curl -X PUT ".../boards/BOARD_ID/prefs/cardCovers?value=false&$AUTH"` — confirm
  with the user first, it's board-wide, not scoped to one card.
- A comment action's text is nested at `.data.text`, not top-level `.text`. Verifying a
  posted comment with `tr_get "/1/actions/ID?fields=text"` returns no `text` key. Use
  `fields=data` and read `.data.text`.
- The Windows non-ASCII argv trap bites **checkItem names too**, not just comments/desc.
  All of `tr_card_create`, `tr_checklist_add`, `tr_checkitem_add`, `tr_checkitem_set` are
  **file-safe** (pass a FILE path for any non-ASCII, `name@FILE`). Pure-ASCII literals are
  fine inline.
- **Batched writes fail — do ONE write per Bash invocation.** Ticking many checkItems
  (or any loop of PUT/POST) in a single `source trello.sh && for ... tr_checkitem_set ...`
  invocation: only the FIRST write lands; the rest return an empty body / `HTTP 000`,
  regardless of `sleep` spacing. GET reads batch fine; it is specifically multiple *writes*
  from one shell process (burst-outbound throttle). Reliable fix: one write per Bash tool
  call (issue them as separate/parallel single-tick calls).

## Card lifecycle (typical flow)
A common pattern: **create a card in an "In Progress" list** with a checklist of action
items -> **tick each item** as done (`tr_checkitem_set CARD ITEM complete` — state=complete
renders the checkmark) -> when all done, **post a completion comment** (`tr_comment`,
team-facing style) -> **move it to a "Review" list** (`tr_card_move CARD REVIEW_LIST`).
Discover your own board/list IDs with `tr_boards` and `tr_lists BOARD_ID`, then record the
ones you use often in project memory so you don't re-fetch them each session.
