# ryan
Skills for Website Rebuilds and SEO, by Uppercut Digital.

| Skill | What it does |
|---|---|
| [`keyword-drop-analysis`](keyword-drop-analysis/SKILL.md) | Works out why a client keyword dropped (Ahrefs Rank Tracker, GSC via SEO Gets, search volume, SERP and competitor comparison) and gives a short, prioritised fix list |
| [`wp-page-build`](wp-page-build/SKILL.md) | Builds and publishes a page into WordPress/Elementor over the REST API, then verifies it landed |

## Use a skill in every Claude session

Upload it to your claude.ai account once. Account skills load in Claude chats, Claude Code on the web, Cowork, the desktop app and the terminal (Claude Code 2.1.273+).

1. Zip the skill folder so the zip contains `keyword-drop-analysis/SKILL.md`:
   `zip -r keyword-drop-analysis.zip keyword-drop-analysis`
2. In claude.ai, open **Settings → Capabilities → Skills** (the menu is called **Customize** in the desktop app), choose **Upload skill** and pick the zip.
3. Make sure the skill is switched on.

To run it, ask in plain English ("why did *suit tailor* drop for Woolcott St?") or type `/keyword-drop-analysis`. If that name clashes with another skill, use `/anthropic-skills:keyword-drop-analysis`.

**After you edit a skill here, upload the new zip again.** The copy in your account doesn't update itself from this repo.

### Frontmatter rules

claude.ai rejects any upload whose `SKILL.md` frontmatter has fields other than `name`, `description`, `license`, `compatibility`, `metadata` and `allowed-tools`. Put anything else (version, category, usage) under `metadata` or in the body. `name` must match the folder name. `description` must be 1,024 characters or fewer, with no `<` or `>`.
