# mail-classifier

A Gmail label bot that runs entirely on a home GPU. n8n polls the inbox, sends
each new message to a local LLM for classification, and applies the matching
Gmail label. No email content ever leaves the network.

    Gmail Trigger (poll, 15 min)
      -> Classify (Ollama, qwen3:14b)
      -> Map Label (code)
      -> Has Label? (if)
          yes -> Apply Label (Gmail)

## Why a local model

The alternative is an API call per email, forever, to a provider that gets to
read the subject line and sender of everything that hits the inbox. Running
the classifier against Ollama on an RTX 3060 costs electricity, not a
per-request fee, and the email body never crosses the LAN boundary.

## The prompt

The classify step sends the sender, subject, and a 400-character snippet to
`qwen3:14b` with a fixed category list (orders, LinkedIn, updates, promo,
security, jobs, 3D-printing) and asks for exactly one code back. `/no_think`
is appended to skip the model's chain-of-thought output, since only the
category code is used. The code node strips any leftover `<think>` block
before matching, in case a model ignores that instruction.

## Reliability

- The classify request retries twice with a 5-second gap and continues the
  workflow on failure rather than blocking the queue, so one slow or dropped
  Ollama response doesn't stall every email behind it.
- Requests batch at size 1 with a 500ms interval between them, so a burst of
  new mail doesn't hit Ollama concurrently.
- If the model's reply doesn't match any known category, `labelId` comes back
  empty and the `Has Label?` branch skips the Gmail call. Unmatched mail is
  left alone instead of getting a wrong label.

## Setup

1. Import `workflow.json` into n8n.
2. Point the `Classify (Ollama)` node's URL at your own Ollama host (currently
   `http://<ollama-host>:11434/api/generate`).
3. Attach your own Gmail OAuth2 credential to the `Gmail Trigger` and
   `Apply Label` nodes. The exported file ships with a placeholder
   credential ID, not a real one.
4. Create the Gmail labels referenced in the `Map Label` node's category map
   (or edit the map to your own label IDs) before turning the workflow on.

## Known limitations

- The category list is fixed in the prompt. Adding a category means editing
  both the prompt and the `Map Label` code, not a config file.
- No logging of what got classified as what. If the model mislabels
  something, there's no history to check beyond Gmail's own label view.
- Polling every 15 minutes means labels lag new mail by up to that long;
  there's no push/webhook trigger from Gmail here.
- Never load-tested past normal personal-inbox volume.

## License

MIT.
