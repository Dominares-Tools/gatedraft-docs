# Changelog

## 1.0.3 (2026-10-06)

Tests only. No code change.

- Your copy's README test file no longer carries two helpers that nothing in it uses. Version number only otherwise.

## 1.0.2 (2026-10-05)

Tests only. No code change.

- Buyer test suite restored; no code change. More of the README-checking tests now ship and run in your copy. Version number only otherwise.

## 1.0.1 (2026-10-04)

- The opt-out gate now fails closed when it is set up wrongly: a path or string passed as the list
  (`Desk` raises `TypeError`; `load_suppressed` refuses), a JSON object with neither an `addresses` nor an
  `email` key, and a file with content but no address in it all refuse every draft with
  `suppression_unavailable`. An empty or comments-only file still means nobody has opted out.
- `Desk.redraft` no longer drafts for a thread filed away (`filed_away`, with the folder) or for machine
  mail (`machine_mail`).
- `Desk.mark_sent` without `sent_at` stamps the reply with the current time, so it sorts after the question
  it answers instead of first.
- Thread rows sort by parsed time, not by text; a row with no readable time keeps its place.
- The `intake --dry-run` now prints the instructions and the earlier turns as well as the final message,
  built from the same state as a real run. `intake` says on stderr when no `--optout` list was given, and
  reads a list of messages with one desk.
- Docs: the classifier files into Newsletters, Removal requests or Do not contact; Junk is a folder a person
  files into. Said plainly what the fail-closed claim covers.

## 1.0.0 (2026-10-04)

First release.

- `Desk.intake`: for one inbound email, run the gates, sort the message, ground and write a reply
  draft, and hold it for a person. Nothing is sent; there is no send function.
- Gates that run before any model is paid: outbound mail, your own mailings, a missing thread key, mail
  too old to answer (fails open), machine mail (bounces, out-of-office, no-reply senders), and the opt-out
  list (fails closed: an unreadable list refuses every draft, with its own reason).
- A four-step, short-circuiting triage classifier: it files into Newsletters, Removal requests or Do not
  contact, or gives tags (Materials, Event, Question, Concern, Complaint, General, Ignore). It reads only what the sender
  wrote, enforces the step order in the parser, and never files over a person's own filing. A removal
  request is filed, never acted on.
- A grounded reply drafter: your search runs before the model, passages are clipped at a boundary that
  says so, the rules ban placeholders, follow-up promises and claimed actions, and a reply cut off at the
  token limit is refused rather than stored.
- `Desk.redraft` (write it again after a note is fixed), `Desk.sort`, `Desk.mark_sent`.
- JSON-file stores for folders, tags, drafts and threads, written atomically and refused (bytes untouched)
  when the file cannot be read; in memory when no path is given.
- Model transports: `openai_chat` (any OpenAI-compatible endpoint), `anthropic_messages`, `canned`. Standard
  library only.
- The `gatedraft` command: `intake` (with `--dry-run`), `classify`, `redraft`, `show`.
- Synchronous API. Python 3.10 or newer, no dependencies.
