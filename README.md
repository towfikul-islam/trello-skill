# Trello skill for Claude Code

A **Claude Code skill** (not an MCP server) that lets Claude talk to your Trello over the
raw REST API with `curl`: list boards / lists / cards, read and **download attachment
bytes**, post comments, upload local files to cards, and drive the create → checklist →
move card lifecycle.

It's just two files — a `SKILL.md` (which Claude auto-discovers) and a `trello.sh` helper
of shell functions. Your API credentials live in a separate private file that is **never**
part of this repo.

> **Skill vs MCP?** A *skill* is a folder Claude Code auto-loads from `~/.claude/skills/`.
> An *MCP server* is a long-running process you register in settings. This is the former —
> nothing to install or keep running, just files to drop in place.

---

## Prerequisites

- [Claude Code](https://claude.com/claude-code) installed.
- `bash`, `curl`, and `python` — all present by default on macOS and Linux.
- A Trello account.

---

## Install

### 1. Put the skill files in place

Clone this repo, then copy the `trello/` folder into your Claude skills directory:

```bash
git clone https://github.com/<your-username>/trello-skill.git
cd trello-skill
mkdir -p ~/.claude/skills
cp -r trello ~/.claude/skills/trello
```

You should now have `~/.claude/skills/trello/SKILL.md` and
`~/.claude/skills/trello/trello.sh`.

### 2. Get your Trello API key + token

Open **https://trello.com/power-ups/admin/new** and fill in the **New App** form. Only
*App name* and *Workspace* matter for personal use; **Email**, **Support contact**,
**Author**, and **Iframe connector URL** can be anything / left blank — the iframe URL at
the bottom is optional. Click **Create**.

![New App form](assets/01-new-app.png)

Creating the app takes you straight to the **API key** page. Click **Generate a new API
key**.

![Generate a new API key](assets/02-generate-key.png)

Confirm in the dialog by clicking **Generate API key**.

![Generate API key confirmation](assets/03-generate-modal.png)

You now have your **API key**. Copy it. Then, to get a **token**, click the **Token** link
in the sentence *"you can manually generate a Token"* (top-right of this page), authorize
the app when Trello prompts you, and copy the token it shows.

![API key and Token link](assets/04-key-and-token.png)

> Keep the **Secret** to yourself — this skill does not need it. You only need the **API
> key** and the **token**.

### 3. Create your private credentials file

Copy the example into your home Claude directory and fill in the two values you just got:

```bash
cp .trello.env.example ~/.claude/.trello.env
# then edit ~/.claude/.trello.env and paste your TRELLO_KEY and TRELLO_TOKEN
```

### 4. Add your member ID

With the key and token set, fetch your member ID and paste it into the same file:

```bash
source ~/.claude/skills/trello/trello.sh
tr_get "/1/members/me?fields=id,username"
```

Copy the `id` from the output into `TRELLO_MEMBER_ID` in `~/.claude/.trello.env`.
(Only `tr_mine` / `tr_mine_board` need it, but it's a one-time step.)

### 5. Verify

```bash
source ~/.claude/skills/trello/trello.sh
tr_me        # -> your username + full name
tr_boards    # -> your boards and their IDs
```

If those return your account, you're done. In a Claude Code session, just mention Trello
(e.g. *"list my Trello boards"*) and Claude will use the skill.

---

## Usage

Each shell the skill runs in is fresh, so it re-`source`s `trello.sh` every call, then uses
functions like:

| Function | What it does |
|---|---|
| `tr_me` | Verify creds → your username |
| `tr_boards` | List boards + IDs |
| `tr_lists BOARD_ID` | Lists on a board |
| `tr_cards LIST_ID` | Cards in a list |
| `tr_card SHORTLINK` | One card by shortlink/ID |
| `tr_atts CARD_ID` | Attachments on a card |
| `tr_dl CARD_ID ATT_ID FILENAME OUTPATH` | Download attachment bytes |
| `tr_comment CARD_ID "text"\|FILE` | Post a comment |
| `tr_upload CARD_ID FILEPATH` | Attach a local file |
| `tr_card_create` / `tr_checklist_add` / `tr_checkitem_set` / `tr_card_move` | Card lifecycle |
| `tr_mine LIST_ID` / `tr_mine_board BOARD_ID` | Cards assigned to you |

The full reference, comment-writing style, safety notes, and hard-won gotchas live in
[`trello/SKILL.md`](trello/SKILL.md).

---

## Security

- Your token grants **full access to your Trello account**. It lives only in
  `~/.claude/.trello.env`, which `.gitignore` keeps out of git. Never paste it into a
  commit, a comment, or a chat.
- This repo contains **no credentials** — only the skill code and this guide.
- Destructive operations (delete/archive) are intentionally *not* wrapped; Claude will
  confirm with you before running them by hand.

---

## Notes

- **Each user brings their own creds.** Nothing about the original author's account
  transfers — you generate your own key, token, and member ID above.
- **Board/list IDs are discovered at runtime** (`tr_boards`, `tr_lists`). Claude records
  the ones you use often in its own memory over time.
- The `SKILL.md` mentions a Windows text-encoding quirk (non-ASCII args getting mangled).
  macOS and Linux are UTF-8 native, so it won't bite you — the skill's file-based text
  path works everywhere regardless.
