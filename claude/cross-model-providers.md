# Cross-model providers (Claude Code adapter)

Shared reference for running one goldfish pass through a **different model provider** instead of a Claude subagent — a genuinely independent perspective, not just a different prompt on the same model. Any `eg-*` command can offer this as an *optional* pass. This file is the single source of truth for the invocation mechanics so each command only needs to reference it, not duplicate `curl`/`jq` blocks.

A cross-model pass is never spawned via the `Agent` tool (that tool only launches Claude subagents) — it's a direct API call via `Bash`. It is always **optional and fail-soft**: missing credentials, network errors, or malformed responses skip the pass with a one-line note; they never stop the command.

## Provider registry

Secrets are read from an environment variable first; `secret-tool` (Linux/GNOME
keyring) is a fallback for shells that don't already export one, not a requirement.
Windows/Git Bash has no `secret-tool` at all — set the env var directly there.

| Provider | What it is | Secret env var (fallback: secret-tool) | Invocation shape |
|---|---|---|---|
| **DeepSeek** | `deepseek-chat` via DeepSeek's OpenAI-compatible chat-completions endpoint | `DEEPSEEK_API_KEY` (fallback: `secret-tool lookup service deepseek account api`) | Single request/response `curl` |
| **Microsoft Copilot** | A published Microsoft Copilot Studio agent, reached over the Direct Line 3.0 protocol | `MS_COPILOT_DIRECTLINE_SECRET` (fallback: `secret-tool lookup service ms-copilot account directline-secret`) | Start conversation → post activity → poll for reply (3 calls) |

Both are chosen by the user per-pass via the `AskUserQuestion` template below — neither runs unattended by default except where a command's own doc says otherwise (currently none; `eg-brainstorm` used to auto-run DeepSeek and now asks too, see its Step 0).

## The "which provider" question template

Drop this into a command's framing step (its own `AskUserQuestion` call) wherever a pass could benefit from an outside model:

```
question: "Should <name of the pass, e.g. 'the diagnosis goldfish' / 'one research lens' / 'one reviewer pass'> also run through an external model for a genuinely independent second opinion?"
header: "Cross-model"
multiSelect: true
options:
  1. **DeepSeek (Recommended)** — "Independent open-weight model, cheap, fast. Good default second opinion."
  2. **Microsoft Copilot** — "Runs the pass through your configured Copilot Studio agent instead of/alongside Claude."
  3. **None** — "Claude-only for this pass."
```

If the user picks one or more providers, run each per the invocation recipes below, **in addition to** the normal Claude goldfish(es) for that pass — never as a replacement, since the whole point is comparing outputs. If a command already spawns a Claude goldfish for the pass, the cross-model call(s) run alongside it (parallelizing via background `Bash` where the harness supports it is a nice-to-have, not a requirement — sequential is fine given each pass has its own timeout).

## Invocation: DeepSeek

```bash
DEEPSEEK_KEY="${DEEPSEEK_API_KEY:-}"
if [ -z "$DEEPSEEK_KEY" ] && command -v secret-tool >/dev/null 2>&1; then
  DEEPSEEK_KEY="$(secret-tool lookup service deepseek account api 2>/dev/null)"
fi
if [ -z "$DEEPSEEK_KEY" ]; then
  echo "DEEPSEEK_PASS_SKIPPED: no key in \$DEEPSEEK_API_KEY or libsecret"
else
  PROMPT_FILE=$(mktemp)
  cat > "$PROMPT_FILE" <<'EOF'
<the filled-in prompt for this pass>
EOF
  jq -n --rawfile p "$PROMPT_FILE" '{model: "deepseek-chat", messages: [{role: "user", content: $p}]}' \
    | curl -sS --max-time 60 https://api.deepseek.com/chat/completions \
        -H "Authorization: Bearer $DEEPSEEK_KEY" \
        -H "Content-Type: application/json" \
        -d @- > /tmp/deepseek_pass_response.json
  rm -f "$PROMPT_FILE"

  if [ $? -ne 0 ] || jq -e '.error' /tmp/deepseek_pass_response.json >/dev/null 2>&1; then
    echo "DEEPSEEK_PASS_SKIPPED: request failed or API returned an error"
    jq -r '.error.message // "no error body"' /tmp/deepseek_pass_response.json 2>/dev/null
  else
    jq -r '.choices[0].message.content' /tmp/deepseek_pass_response.json
  fi
fi
```

## Invocation: Microsoft Copilot (Copilot Studio, Direct Line)

Requires a published Copilot Studio agent with a Direct Line secret. As of current
Copilot Studio, this is **not** in the Channels gallery at all (there's no "Custom
channel"/"Mobile app" tile issuing one) — it's under **Settings → Security → Web
channel security**. Secret 1/Secret 2 are generated there by default and work as a
Direct Line bearer credential immediately; the "Require secured access" toggle on
that same page does not need to be on for a valid secret to authenticate — it only
additionally blocks unsecured/no-secret access (e.g. the Demo website). Store the
secret in the `MS_COPILOT_DIRECTLINE_SECRET` environment variable (or libsecret as
`service=ms-copilot account=directline-secret` if your shell already uses it). This
is a 3-call conversation flow, not a single request/response — Copilot Studio
replies asynchronously, so the script polls briefly.

```bash
COPILOT_SECRET="${MS_COPILOT_DIRECTLINE_SECRET:-}"
if [ -z "$COPILOT_SECRET" ] && command -v secret-tool >/dev/null 2>&1; then
  COPILOT_SECRET="$(secret-tool lookup service ms-copilot account directline-secret 2>/dev/null)"
fi
if [ -z "$COPILOT_SECRET" ]; then
  echo "COPILOT_PASS_SKIPPED: no key in \$MS_COPILOT_DIRECTLINE_SECRET or libsecret"
else
  # 1. Start a Direct Line conversation
  CONV=$(curl -sS --max-time 30 -X POST https://directline.botframework.com/v3/directline/conversations \
    -H "Authorization: Bearer $COPILOT_SECRET")
  CONV_ID=$(echo "$CONV" | jq -r '.conversationId // empty')
  CONV_TOKEN=$(echo "$CONV" | jq -r '.token // empty')

  if [ -z "$CONV_ID" ]; then
    echo "COPILOT_PASS_SKIPPED: could not start Direct Line conversation (bad secret or channel not published)"
  else
    # 2. Post the prompt as a user activity
    PROMPT_FILE=$(mktemp)
    cat > "$PROMPT_FILE" <<'EOF'
<the filled-in prompt for this pass>
EOF
    jq -n --rawfile p "$PROMPT_FILE" '{type: "message", from: {id: "eg-goldfish"}, text: $p}' \
      | curl -sS --max-time 30 -X POST "https://directline.botframework.com/v3/directline/conversations/$CONV_ID/activities" \
          -H "Authorization: Bearer $CONV_TOKEN" \
          -H "Content-Type: application/json" \
          -d @- > /dev/null
    rm -f "$PROMPT_FILE"

    # 3. Poll for the bot's reply (Copilot Studio responses can take several seconds)
    REPLY=""
    for _ in 1 2 3 4 5 6; do
      sleep 5
      ACTIVITIES=$(curl -sS --max-time 30 "https://directline.botframework.com/v3/directline/conversations/$CONV_ID/activities" \
        -H "Authorization: Bearer $CONV_TOKEN")
      REPLY=$(echo "$ACTIVITIES" | jq -r '[.activities[]? | select(.from.id != "eg-goldfish" and .type == "message")] | last // empty | .text // empty')
      [ -n "$REPLY" ] && break
    done

    if [ -z "$REPLY" ]; then
      echo "COPILOT_PASS_SKIPPED: no reply from Copilot Studio within the polling window"
    else
      echo "$REPLY"
    fi
  fi
fi
```

## On success

Treat the provider's output exactly like a Claude goldfish's for that pass, but tag it `[lens: Cross-model — DeepSeek]` or `[lens: Cross-model — Microsoft Copilot]` (whichever ran) instead of a Claude lens name, so synthesis steps can call out cross-model agreement/disagreement specifically. A different model reaching the same conclusion as the Claude pass(es) is a stronger convergence signal than another same-model pass agreeing; a different model landing somewhere else is the cheapest anchoring check available.

## On failure

Skip that provider's pass, do **not** stop the command, and carry a one-line note forward for the command's final report — `<Provider> pass unavailable: <reason>`. The Claude-only output is always sufficient to proceed without it.
