---
name: linkedin-post
description: Draft a result-driven LinkedIn post + AI-generated tree architecture diagram, then publish to your personal LinkedIn profile via browser automation. Opinionated structure (hook on WHY → OUTCOME → HOW → open question + hashtags). Uses ChatGPT.com (via playwright with your authenticated session) for content + image gen. Uses LinkedIn.com (via playwright) for publishing. Three modes — full pipeline, draft-only, publish-only. Use when the user asks to "post to LinkedIn", "share on LinkedIn", "write a LinkedIn post", or announce a project/build/win to a professional audience.
---

# /linkedin-post — Reusable LinkedIn posting workflow

End-to-end skill for builder-voice LinkedIn posts with optional architecture diagrams. Built from a battle-tested run on 2026-05-13, extended 2026-05-19 with the codebase-research step. The browser automation paths are validated against current chatgpt.com and linkedin.com DOM.

## Prerequisites

- `mcp__plugin_playwright_playwright__*` tools must be loaded (this is the playwright MCP). If unavailable, fall back to delivering files for manual posting.
- The user must have a ChatGPT Plus/Pro/Team subscription (image generation is paywalled). Free-tier login still works for text but no image gen.
- The user must have a LinkedIn account. Auto-sign-in via Google works if the playwright browser hits LinkedIn while a Google session exists in cookies. Otherwise the user logs in manually in the playwright window.

## Mode detection (Step 0)

Parse the user's invocation. Modes:

| Invocation | Mode | What runs |
|---|---|---|
| `/linkedin-post` (no args) | **full** | Brief gathering → draft → diagram → review → publish |
| `/linkedin-post draft` or `... draft <topic>` | **draft** | Brief gathering → draft + diagram → save to disk → STOP |
| `/linkedin-post publish --file X --image Y` | **publish** | Skip drafting → open linkedin.com → fill composer with existing files → publish |
| `/linkedin-post <natural-language topic>` | **full** | Use the trailing text as the topic seed; skip the first brief question |

For natural-language invocations like "post about my new caching layer to LinkedIn" → infer **full** mode with topic seed.

## The opinionated post structure (apply in all modes)

The actual distilled value of this skill. Every post follows:

1. **Hook line** (1 sentence). A concrete result, a contrarian observation, or a specific number. NEVER start with "I'm excited to share" or "Today I'm announcing".
2. **The WHY** (2–4 short paragraphs). The real problem the person was facing before this thing existed. Make it tangible — name the pain, the cost, the failure mode.
3. **The OUTCOME** (1–3 lines). A specific concrete win with a number or time: "diagnosed in 90 seconds", "cut weekly toil from 6 hours to 12 minutes", "found three bugs the test suite missed". This is the load-bearing line of the post.
4. **What I built** — emoji bullets, one per component. Use these glyphs: 📊 🤖 ⚡ 🔧 🚀 🎯 🔍 📦 🧩 ⚙️. One emoji per bullet, no double-emoji. 3–6 bullets max.
5. **Brief HOW operating sentence** — "every morning a cron writes X", "the agent reads from Y", etc. One paragraph.
6. **Open question** to peers. Specific, not generic. "What workflow do you wish was automated?" beats "What do you think?"
7. **Hashtags** — 3–5, trending in the target community. Defaults for tech-founders: `#BuildInPublic #Shopify #GoogleAds #Ecommerce #PaidAcquisition` — but customize per topic.

**Word count:** 180–230. LinkedIn truncates around line 3 in the feed — front-load the hook + OUTCOME.

**Banned vocabulary** (rewrite if drafter outputs any): delve, leverage, unlock, transformative, comprehensive, robust, nuanced, multifaceted, furthermore, moreover, pivotal, landscape, tapestry, underscore, foster, showcase, intricate, vibrant, fundamental.

**Banned punctuation:** em dashes (—), exclamation points outside quoted speech.

**Allowed emojis:** in bullets only, never in body prose, never in hashtags.

## Codebase research (when the post describes real repo work)

When the post is about something the user actually built in a repo — they name a repo, a branch, a feature, or say "review the work in that branch" — do NOT draft from the user's narrative alone. The spoken narrative is the *marketing angle*; the repo is the *ground truth*. A diagram or a bullet list built only from the narrative will be plausibly but specifically wrong (wrong component names, wrong data flow, invented services).

Run this BEFORE generating the diagram, and ideally before finalizing the post's bullets:

1. **Locate the repo and branch.** `git -C <repo> branch --show-current`, then `git log --oneline -30 | grep -iE "<feature-keyword>"` to find the feature's commits.
2. **Read the commits.** `git show --stat <sha>` on the substantive ones — commit bodies often explain intent, trade-offs, and gotchas.
3. **Read the actual source.** Find the feature's service/module directory; read each file's header docblock (they usually spell out intent, the data shape, and the data flow). 20 lines per file is enough.
4. **Read the feature's docs.** CHANGELOG entry, any spec or plan markdown.
5. **Write a 5–10 line summary of the REAL architecture:** entry point, the hot path, the data flow, each component and what it does, and any cold/offline/calibration paths. Show this summary to the user.
6. **Ground the diagram and fact-check the post on this summary** — real component names, real flow, real direction of the accent/write-back path.

This costs a handful of tool calls and turns a generic guess into a diagram that matches the merged code. Always worth it when a repo exists. Skip it only for conceptual posts with no underlying codebase.

## Diagram opinions

When the user wants a visual (default: ask), the diagram should be:

- **Accurate to the code, when there is code.** If the Codebase research step ran, the diagram must reflect the *real* architecture — real component names, real entry point, real data flow, including any cold/calibration path. A generic tree built from the verbal pitch is the most common way this skill ships a wrong diagram. A conceptual hub-and-spoke is acceptable only for posts with no underlying repo.
- **Tree top-down** layout (not centered hub-and-spoke). Top node = the agent/system. Branches = the integrations/services. Sub-branches = the actions per service. Optionally a bottom node = the output.
- **Real service logos** for each node — let the image model render actual Google Analytics, Google Ads, Shopify, PostgreSQL, Anthropic etc. logos. Specify in prompt.
- **One accent color** for the most important / write-back path (e.g., orange border on the service the agent mutates).
- **16:9 landscape**, clean modern style, flat geometric, navy + accent palette, white/light-gray background.
- **No fake metrics**, no decorative icons that aren't structural, no faces/people, no 3D, no photorealism.

## Browser automation paths

### ChatGPT.com (content + image generation)

1. `browser_navigate("https://chatgpt.com/")`.
2. `browser_snapshot()` — verify logged-in by looking for the profile button. If you see "Log in" / "Sign up" buttons, ask user to log in in the playwright window, then re-snapshot.
3. **Switch to Thinking model** — click the "Instant" model button, then click the "Thinking • Extended" option. The testid for the option button has been `model-switcher-gpt-5-5-thinking` (will change as models advance — fall back to clicking the menu item whose label contains "Thinking").
4. Type the **post prompt** into the chat textbox (`role: textbox, name: "Chat with ChatGPT"`), submit. Wait ~90–120s for Thinking + image gen if requested in same turn.
5. **Multi-turn requests can collapse** — if you ask for "post + diagram" in one message with Thinking model, the model often only delivers ONE of them. Always send the post draft and the diagram in **separate turns**.
6. After image gen, the response contains a button labeled `"Generated image: <name>"` — clicking it opens a **fullscreen lightbox**. Screenshot the `img` inside the lightbox for the cleanest result (no chatgpt UI artifacts).
7. Image text quality is the failure mode. GPT-image often misrenders short text like "ANTHROPIC" → "ANTHROP\C". Acceptable at LinkedIn thumbnail size; surface to user but don't block.

### LinkedIn.com (publish)

1. `browser_navigate("https://www.linkedin.com/")`. If Google session cookies exist, auto-signin can take 5–10s and you land at `/feed/`.
2. If still on the marketing homepage, ask user to log in manually in the playwright window.
3. From `/feed/`, snapshot, find button `"Start a post"`, click.
4. Wait ~2s. Snapshot. Find the composer dialog (`dialog "Create post modal"`) and the editor (`textbox "Text editor for creating content"`). Snapshot files can exceed token limits — pass `filename: ".playwright-mcp/snap-<phase>.md"` and grep instead of inlining.
5. `browser_type` into the editor with the post body. LinkedIn auto-bolds hashtags.
6. Click `button "Add media"` → triggers file chooser modal state. Call `browser_file_upload(paths: [<absolute path to PNG>])`.
7. Wait ~3s for upload. Snapshot. Click `button "Next"` (this is the image-editor's confirm; without it, the final Post button does not enable).
8. Verify composer state: author shows "Post to Anyone", text fully present, image preview visible. Screenshot the dialog for user verification.
9. **FINAL APPROVAL GATE** — call AskUserQuestion. The "Post" action is hard-to-reverse (you can delete after but early-feed viewers will have already seen it). Options: post now / schedule / edit / cancel.
10. On approval: click `button "Post"` (exact match — there's also "Schedule post" nearby; use `name: 'Post', exact: true` in selector).
11. Wait ~4s. Grep the new snapshot for `"Post successful"` and the share URL: `https://www.linkedin.com/feed/update/urn:li:share:NNNNNNNN/`.

## Step-by-step workflow

### Mode: `full` (default)

1. **Brief gathering** — AskUserQuestion. Minimum questions:
   - What's the topic / what did you build?
   - What was the real problem (one sentence)?
   - What's the concrete win / outcome with a number?
   - Tech stack used (3–6 components)?
   - Want an architecture diagram? (yes/no)
   - Target audience (founders / engineers / industry-specific)?
   Skip questions whose answers are already obvious from the user's invocation message — don't make them repeat themselves.

2. **Save the brief** to `/tmp/linkedin-brief-<timestamp>.md` so it's not lost mid-flow.

3. **Codebase research** — if the post describes real repo work, run the **Codebase research** section above. Fold the architecture summary into the brief file. Skip only for conceptual posts with no repo.

4. **Generate post text** — by default, use chatgpt.com browser path (Thinking model, separate turn from image). If user has codex CLI working and not over usage limit, that's also valid. Build a focused prompt embedding the brief + the opinionated structure rules + the banned-vocabulary list.

5. **Generate diagram** (if requested) — second chatgpt.com turn. Specify the real architecture from step 3 (or a tree top-down for conceptual posts), real logos for the named services, orange accent on the write-back/calibration path, 16:9, no clutter.

6. **Save both artifacts** to a working directory the user can find — recommend `.playwright-mcp/linkedin-post-<topic-slug>.md` and `.playwright-mcp/linkedin-diagram-<topic-slug>.png`.

7. **Show user** the post text inline (markdown) and the diagram (Read the PNG). AskUserQuestion: approve / tweak text / regenerate diagram / cancel. If the user asks to regenerate the diagram with a different layout, see the image-edit-stickiness failure mode — do it in a fresh chat.

8. **On approve** → switch to publish phase.

### Mode: `publish` (or last step of `full`)

1. Verify `--file` and `--image` (or in full mode, the artifacts from step 6) exist.
2. Navigate to `https://www.linkedin.com/`, handle login.
3. Click "Start a post". Type text. Add media. Next.
4. Screenshot composer. AskUserQuestion final approval.
5. Click Post. Capture success URL.
6. Tell the user the URL.

### Mode: `draft` (stops before publish)

Steps 1–7 of full mode, but skip step 8. Tell user: "Drafted. To publish later: `/linkedin-post publish --file <path> --image <path>`."

## Failure modes + fallbacks

| Symptom | Cause | Fallback |
|---|---|---|
| chatgpt.com shows "Log in" buttons | Playwright browser is isolated, no cookies | Ask user to log in in the playwright window. Do not attempt to drive the login form. |
| ChatGPT response says "You've hit your usage limit. Upgrade to Pro..." | Subscription rate-limited | Offer: wait until reset / user pastes API key / agent drafts manually using conversation context. |
| GPT generates only image, not text (or vice versa) | Multi-task collapse in Thinking model | Send the missing piece as a follow-up turn. |
| ChatGPT re-renders the OLD diagram when asked to regenerate with a new layout | In-conversation image gen *edits* the prior image; a same-thread "redo with this layout" anchors to the old composition and ignores new layout instructions | Start a FRESH chat (`browser_navigate("https://chatgpt.com/")`) and generate the diagram standalone with the full prompt. Never iterate layout changes in the same thread. Minor tweaks (recolor, fix a typo) are fine to do in-thread; structural layout changes are not. |
| Screenshot of a "Generated image" opens/captures the wrong (older) image | Multiple generated images in one chat share the same accessible name; `getByRole('button', {name: 'Generated image: ...'})` matches the FIRST one | Re-snapshot, take the exact `ref` of the LATEST `button "Generated image: ..."`, and target that ref. Or just use a fresh chat so there is only one image. |
| `browser_take_screenshot` errors "File access denied: /tmp/..." | Playwright MCP has restricted roots | Save to `.playwright-mcp/` instead — always inside the allowed root. |
| `browser_snapshot()` result exceeds token limit | Page is huge (LinkedIn feed is) | Pass `filename: ".playwright-mcp/snap-N.md"` and grep the file for the elements you need (e.g., `grep -nE "Start a post\|textbox\|Post\""`). |
| LinkedIn auto-signin lands on a security check | Google account isn't already trusted | Ask user to complete the check in the playwright window, wait for `/feed/` URL, then continue. |
| LinkedIn "Post" button stays disabled after typing | The hidden `Next` step after image upload wasn't clicked | Click `button "Next"` first. |
| Composer DOM changed (refs not matching) | LinkedIn changed selectors | Re-snapshot, search by ARIA role + accessible name (`role: button, name: "Start a post"`), not by ref. |

## Approval gates (NEVER skip)

This skill publishes to the user's real LinkedIn feed and consumes ChatGPT credits. Always pause for explicit approval at:

1. **Post text approved** — before generating diagram (saves image-gen credit if text is wrong).
2. **Both artifacts approved** — before opening linkedin.com.
3. **Final Post confirmation** — after composer is loaded, screenshot the composer, then AskUserQuestion. Include the live URL pattern in the success message so the user can click immediately.

Hard rule: never click the final "Post" button without an explicit approval in the immediately-preceding AskUserQuestion answer.

## What this skill is NOT

- Not a LinkedIn API client. For API-based posting (no browser), see `scripts/linkedin-post.mjs` if it exists in the project (built 2026-05-13 in turn14-integration). That requires LinkedIn Developer app + OAuth setup.
- Not a scheduler. For scheduling, the workflow currently delegates to LinkedIn's native "Schedule post" UI (clock icon next to Post). A future version may integrate Zoho Social MCP `createSocialSchedule`.
- Not a content polisher. The first chatgpt.com draft is usually 90% there — the value is the structure, the bans, and the publishing pipeline. Don't fall down a rewrite rabbit hole; if the second draft isn't right, ask the user.

## Artifacts to leave behind

After a successful post, the workspace should contain:

- `.playwright-mcp/linkedin-post-<slug>.md` — final published text
- `.playwright-mcp/linkedin-diagram-<slug>.png` — final image
- `.playwright-mcp/linkedin-composer-preview.png` — what the user saw before clicking Post

These are scratch files (most projects have `.playwright-mcp/` in `.gitignore` or ignored by being under `.claude/worktrees/`). Offer to clean them up at the end. The published share URL is also worth saving — append to user's notes / project CHANGELOG if they want.

## Voice guidance for the drafter prompt

Always include this verbatim block in the chatgpt.com prompt:

> Voice: technical founder building in public. Tone: professional, builder-talking-to-builders. NO em dashes. NO exclamation points outside quoted speech. NO emojis except as bullet markers (one per bullet, max 6 bullets). NO AI vocabulary: delve, leverage, unlock, transformative, comprehensive, robust, nuanced, multifaceted, furthermore, moreover, pivotal, landscape, tapestry, underscore, foster, showcase, intricate, vibrant. Open with a concrete result or contrarian observation, NOT "I'm excited to share." Word count target: 180–230. Write ONLY the post body, ready to copy-paste into LinkedIn — no headline, no commentary, no "Here's a draft."

## Telemetry

If you discover a new failure mode while running this skill (selector changes, model name changes, new LinkedIn UI flows), update the **Failure modes** or **Browser automation paths** sections of this file directly before completing the run. The skill should improve with each invocation.
