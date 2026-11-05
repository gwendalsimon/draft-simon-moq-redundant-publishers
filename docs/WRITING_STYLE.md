# Writing Style for draft-simon-moq-redundant-publishers

All authors follow these rules, whether they write by hand or with an AI assistant.
The goal is one voice across sections written by different people. When a rule makes
a sentence harder to read, clarity wins.

## Structure

- Put the conclusion first. A section states its rule, then explains it.
- Open each paragraph with a topic sentence that a reader understands on its own,
  without the previous paragraph. Keep one idea per paragraph.
- Introduce every list or table with a full sentence ending in a colon. Keep list items
  parallel: all full sentences, or all noun phrases.
- Use noun phrases for section titles ("Relay Behavior"), never "How X does Y".

## Sentences

- Write one independent statement per sentence. Do not join two statements that could
  stand alone with ", and", ", but", ", so", or a semicolon. When in doubt, split.
- Use the active voice and name the actor: the Original Publisher, the relay, the
  subscriber, or the endpoint.
- Do not open a sentence with a bare "This" or "That". Attach a noun ("this rule").
- Do not use "X rather than Y" or "X, not Y" to emphasize a point. State the point.
- Do not address the reader as "you". Use a role ("the operator") or "we".

## Words

- Use plain, common words. Many readers are not native English speakers.
- Avoid vague quantifiers and hedges: "some", "many", "several", "significantly",
  "arguably", "simply", "just". Give a number, a reference, or nothing.
- Avoid "whichever", "whatever", and "whenever". Write "regardless of …".
- Use American spelling, as {{MOQT}} does ("behavior", "color").
- Use the Oxford comma: "a, b, and c".
- Use "that" for restrictive clauses (no comma) and ", which" for non-restrictive ones.
- Do not use em-dashes. Use parentheses or a new sentence.
- Use role words as adjectives: "main publisher", "backup publisher", never "the main"
  or "a backup" alone.

## IETF Conventions

- Use BCP 14 keywords (MUST, SHOULD, MAY) in capitals only for normative requirements.
  Write one requirement per sentence, with an explicit actor.
- Use MOQT terms exactly as {{MOQT}} defines them, with its capitalization: Track,
  Group, Subgroup, Object, Track Property, Setup Option, Original Publisher. Write
  "relay" and "subscriber" in lower case, as {{MOQT}} does.
- Define every new term in "Conventions and Definitions" before using it.
- Cite {{MOQT}} for behavior that MOQT already specifies. Do not restate it.
- Write code points as TBD until IANA assigns them. Give message and property
  layouts in the notation of {{MOQT}}.
- Wrap source lines at 80 characters.

## Working with an AI Assistant

- Ask the assistant to read this file before it writes or edits text.
- Never let it invent references, URLs, section numbers, or quotations. Check every
  citation against the source.
- Keep edits minimal. Do not reword text outside the section being changed.
- Do not restore text that another author removed.
