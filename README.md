# gatedraft — Reply drafts from your notes, never sent

**Paid tool: $149 once (plus sales tax where it applies), then a private GitHub repository you're invited to, with every update. Buy: https://dominares.org/tools/gatedraft**

Commercial license v1.3 — see https://dominares.org/license · Support: support@dominares.org

For one inbound email, gatedraft runs the gates, sorts it, searches your own knowledge and writes a reply draft that waits for a person to read it. The gates run before the model is paid. An address on your opt-out list is refused, and a list that can't be read stops every draft to an address. There is no send function in the package.

## Get it

Buy gatedraft at https://dominares.org/tools/gatedraft: one payment of $149, plus sales tax where it applies, and every future update. After checkout you are invited to a private GitHub repository with the source and every version published to it. The repository includes the gatedraft Python package and command, runnable examples and the full test suite.

Requirements: Python 3.10 or newer. Your own model key (OpenAI-compatible or Anthropic) for live drafts; the dry run and canned-reply modes need none.

This page documents version 1.0.3. Install commands and file paths in the documentation below run inside the private repository you are invited to; the source is not in this repository.

## Documentation

### What it does

For one inbound email, in this order:

1. **Gates, before any model is paid.**
   - *Direction.* Your own outbound mail is never drafted for (`outbound`).
   - *Mailing.* A message marked as your own mailing to a list is never drafted for (`blast`).
   - *Age.* Mail older than 30 days (yours to change, or to turn off) is skipped (`too_old`). This one
     fails *open*: a timestamp it cannot read does not stop a draft.
   - *Machine mail.* Bounces, out-of-office replies and no-reply senders are skipped (`machine_mail`).
     It reads only what the sender wrote, never the quoted history under it.
   - *Opt-outs.* An address on your opt-out list is refused (`opted_out`). This one fails *closed*: if
     the list cannot be read, **no one with a readable address** is drafted for (`suppression_unavailable`).
     A list that is set up wrongly counts as unreadable: a path or string given where a list belongs
     (`Desk` raises `TypeError`), a JSON object with neither an `addresses` nor an `email` key, or a file
     with content but no address in it. A sender with no address at all passes this gate, because there is
     nothing to check. With no list given (no `--optout`, or `suppression=None`) there is **no opt-out
     gate**.
2. **Sort.** A four-step classifier files the message in a folder (Newsletters, Removal requests or Do not
   contact: steps 1-3) or gives it tags (Materials, Event, Question, Concern, Complaint, General, Ignore:
   step 4). Junk is a folder a person files into with `desk.folders.set(...)`; the classifier never picks
   it, but a thread there is not drafted for either. Mail filed away is not drafted for (`filed_away`). Nothing here removes anyone from a list: a removal
   request is *filed*, with the sender's own words beside it, for a person to act on.
3. **Ground.** Your `search_fn` (or the built-in keyword search over a folder of notes) is called with the
   subject and what the sender wrote *before* the model is. If that finds nothing it is asked once more
   with the whole message, quote included, because a two-word reply ("yes please") is only answerable from
   what it replies to. The passages it returns are put in the prompt as "the only facts you may use",
   clipped at a sentence boundary that says it clipped. The model is sent the whole message, quote
   included, so it can see what the person is replying to. If nothing is found the prompt says so, and
   the model is told to give a contact instead of inventing an answer.
4. **Draft.** The model writes the reply under rules that ban placeholders like `[TODO]`, promises that
   someone will follow up, and claims that something was cancelled or removed. A reply that hit the token
   limit is refused rather than stored.
5. **Hold.** The draft is saved for a person. There is no send function anywhere in the package.

Every refusal has a named reason, so an empty box on screen is never ambiguous. A model outage is recorded
as a fixed sentence (the provider's own error text goes to the log, not to the screen) and never raises.

### Use it from Python

```python
from gatedraft import Desk, FileSuppression, FolderStore, DraftStore, KeywordSearch, openai_chat

desk = Desk(
    call_fn=openai_chat(),                       # or anthropic_messages(), or any function you write
    model="your-model-name",
    search_fn=KeywordSearch.from_folder("notes/"),   # or your own: str -> list[str]
    suppression=FileSuppression("optout.txt"),       # an unreadable list = no drafts (leave it out = no gate)
    facts="Main email: help@example.org",            # verified contact details the draft may quote
    org_name="Example Co",                           # the sign-off
    folders=FolderStore("state/folders.json"),
    drafts=DraftStore("state/drafts.json"),
)

result = desk.intake(row_id="m-1", thread_key="cid:abc", subject="Re: Friday visits",
                     body=email_text, from_addr="pat.doe@example.com", from_name="Pat Doe")
# {"drafted": True, "reason": ""}   or   {"drafted": False, "reason": "opted_out"}

desk.drafts.get("cid:abc")["text"]     # the draft, waiting for a person
desk.redraft("cid:abc")                # write it again, after you fix a note (still honours the gates)
desk.mark_sent("cid:abc", sent_text)   # record that a person sent it (stamped now), and clear the draft
```

`examples/app.py` is a complete runnable version with a stand-in model.

`Desk` options: `search_fn`, `suppression` (an object with `load()`, a function, or a set of addresses),
`folders`, `labels`, `drafts`, `threads` (all in memory unless you give them a path), `model`,
`classify_call_fn` and `classify_model` (a cheaper model for sorting), `classify=False` (draft for everything),
`org_name`, `facts`, `machine_fn`, `visible_fn`, `max_age_days` (30; `0` turns the gate off),
`budget_errors` (exception types that mean "stop spending": intake ends with reason `budget` and stores
nothing), `draft_max_tokens`, `classify_max_tokens`.

The pieces work on their own, too: `classify()`, `parse_result()`, `build_draft_prompt()`, `draft_reply()`,
`ground()`, `may_draft_for()`, `too_old_to_draft()`, `looks_automated()`, `visible_text()`.

A model transport is any function taking `system`, `messages`, `model` and `max_tokens` and returning the
text, or `(text, incomplete)`. `openai_chat()` talks to any OpenAI-compatible `/chat/completions` endpoint
and `anthropic_messages()` to the Anthropic Messages API, both with the standard library only.

### The command

```text
gatedraft intake MAIL.json [--dry-run] [--state DIR] [--optout FILE] [--knowledge DIR]
                           [--org NAME] [--facts TEXT|FILE] [--max-age-days N]
                           [--provider openai|anthropic --model NAME [--base-url URL]]
                           [--answer-file FILE] [--verdict-file FILE]
gatedraft classify MAIL.json [--dry-run] [--provider ... --model ... | --verdict-file FILE]
gatedraft redraft THREAD_KEY --state DIR [model options]
gatedraft show --state DIR [THREAD_KEY]
```

`MAIL.json` is one message or a list: `subject`, `body`, `from_addr`, and optionally `from_name`, `id`,
`thread_key` or `conversation_id`, `received_at`, `direction`, `section`. See `examples/`.

- A live model needs `--provider` and `--model`, and `GATEDRAFT_API_KEY` in the environment (the OpenAI
  provider also reads `OPENAI_API_KEY`, the Anthropic one `ANTHROPIC_API_KEY`).
- `--answer-file` and `--verdict-file` stand in for the model with a fixed reply, so you can watch the
  whole pipeline (gates, filing, storing the draft) with no key. The draft you see then is the file's
  text, not a model's.
- Without `--verdict-file` or a provider, `intake` does not sort the message and drafts for everything
  that passes the gates.
- Exit status: `0` drafted (or, with `--dry-run`, allowed), `1` refused or failed to draft, `2` the
  command was used wrongly.
- `--optout` is one address per line (`#` comments allowed) or a JSON list (or an object with an `addresses`
  or `email` key). A missing or unreadable file, or one with content but no address in it, refuses every
  draft. Leave `--optout` out and there is no opt-out gate; the command says so on stderr.

### Limits

- **It writes drafts; a person sends them.** Nothing here sends mail, and it does not decide that a draft is
  right. Read every one. The prompt rules reduce invented facts and false promises; they cannot prove a
  model obeyed them.
- **It does not satisfy the law for you.** The opt-out gate honours the list you give it. Getting and recording
  consent, and what the law asks of your mail, stay with you. It does not add opt-out wording to anything.
- **Inbound mail is untrusted text.** A message can try to steer the model, which is one more reason a
  person reads every draft before it goes.
- **The model and the knowledge are yours.** It ships no model and no knowledge. Draft quality depends on
  what your `search_fn` returns, and a model sees the text of the email and the passages you ground it in.
- **The built-in search is a keyword search.** It ranks passages by shared words (with a bonus for a
  passage's first line). It does not understand meaning, so "when are you open" will not find a passage
  that only says "hours". Pass your own `search_fn` for anything serious.
- **Quote-cutting is simple.** `visible_text` cuts at the common reply markers and flattens simple HTML. For
  Outlook headers collapsed onto one line, other languages or nested HTML quotes, pass a better
  `visible_fn` (the sibling tool `replysplit` is built for that).
- **Machine-mail detection reads the subject, the sender's own words and the sender address.** It does not
  read headers such as `Auto-Submitted` or `List-Unsubscribe`; pass `machine_fn` for that.
- **The classifier and the drafter are English-language prompts.** The traps they guard against are
  English patterns.
- **The tag list is fixed** at the seven above, and the folders at five (Inbox, Newsletters, Removal requests,
  Do not contact, Junk).
- **Opt-outs match whole addresses**, case-insensitively, with display names removed. There are no
  domain wildcards and no plus-address folding.
- **The JSON-file stores suit one machine.** Writes are atomic and, on systems with `fcntl`, advisory-locked across processes, but
  they rewrite the whole file; for many machines, give the `Desk` stores of your own with the same methods.
- **`redraft` is a library call, not a button.** It refuses mailings, threads filed away and machine mail,
  and honours the opt-out list. It does not apply the age gate.
- **Thread order is by time.** Dates are read as ISO 8601 or RFC 2822; a row whose date cannot be read stays
  behind the row before it.
- **The age gate and the opt-out gate see the message as it arrives.** A person who opts out after their
  draft was written is caught by `redraft`, not by the draft already waiting; clear it with
  `desk.drafts.clear_draft(thread_key)` when you record the opt-out.
- **Python 3.10 or newer. No dependencies.** The two network transports use `urllib`.

### Development

```bash
python3 -m pytest
```

The tests use stand-ins for every model and a local server on `127.0.0.1` for the network transports: no
network, no account, no key.

---

Docs mirror of the private repository Dominares-Tools/gatedraft; updated with each release.
