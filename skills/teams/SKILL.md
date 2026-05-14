---
name: teams
description: |
  Drive Microsoft Teams via the Microsoft Graph CLI (`mgc`) — list joined teams, list channels, read/post channel messages, read/post 1:1 and group chat messages, fetch the activity feed, and send Adaptive Card payloads. Use when the user says "post to the general channel", "DM Alice on Teams", "what was said in the deploys channel today", "send a card to the team", "list my Teams", or wants to script Teams without the desktop client. This is the headless Teams skill — UI-only flows (joining a call, raising hand) need the desktop app.
metadata:
  openclaw:
    emoji: "💬"
    os: ["win32"]
    requires:
      bins: ["pwsh", "mgc"]
    install:
      - id: "mgc-download"
        kind: "download"
        os: ["win32"]
        url: "https://github.com/microsoftgraph/msgraph-cli/releases/download/v1.9.0/msgraph-cli-win-x64-1.9.0.zip"
        archive: "zip"
        extract: true
        targetDir: "mgc"
        bins: ["mgc"]
        label: "Download Microsoft Graph CLI (mgc) for Windows"
---

# Microsoft Teams (via Microsoft Graph CLI)

Use `mgc` to drive **Microsoft Teams** through Graph: enumerate teams and channels, post messages, send Adaptive Cards, read the activity feed.

## When to Use

✅ **USE this skill when:**

- User says "post a message in the X channel" / "DM Alice on Teams"
- User says "what was discussed in <channel> today"
- User wants to send an **Adaptive Card** to a channel or chat
- User wants to enumerate teams/channels for an inventory script
- Workflow needs to run without the Teams desktop client

## When NOT to Use

❌ **DON'T use this skill when:**

- User wants to **start/join a call** or share screen — Graph does not drive the call UI; use the Teams desktop client
- User wants Slack → `slack` skill
- User wants Discord → `discord` skill
- User wants Telegram → `message` tool with `channel:telegram`
- Bulk-message a whole tenant — that needs an org-wide approved app + admin consent, not interactive `mgc login`

## Setup

`mgc` is the official Microsoft Graph CLI. There is **no winget package**. Install from GitHub releases:

```powershell
$dst = "$env:LOCALAPPDATA\Programs\msgraph-cli"
New-Item -ItemType Directory -Force -Path $dst | Out-Null
$zip = "$env:TEMP\msgraph-cli-win.zip"
Invoke-WebRequest `
  "https://github.com/microsoftgraph/msgraph-cli/releases/download/v1.9.0/msgraph-cli-win-x64-1.9.0.zip" `
  -OutFile $zip
Expand-Archive $zip -DestinationPath $dst -Force
$env:Path = "$dst;$env:Path"
mgc --version
```

### Sign in

```powershell
mgc login --scopes `
  "Chat.ReadWrite Chat.Create ChannelMessage.Send ChannelMessage.Read.All Channel.ReadBasic.All Team.ReadBasic.All TeamsActivity.Read User.Read"
```

- `ChannelMessage.Send` is the **send-to-channel** scope; `ChannelMessage.Read.All` is the matching **read** scope for channel messages.
- `Chat.ReadWrite` covers reading and posting in existing 1:1 / group chats; `Chat.Create` is required to create a new chat (e.g. starting a fresh 1:1).
- Many tenants restrict `Channel.ReadBasic.All` and `ChannelMessage.Read.All` to admin-consented apps. Re-run `mgc login` after admin grants consent.
- Tenant pinning: `--tenant-id <guid|domain>`. Multi-account: re-run `mgc login` per account (only one cached token at a time); scopes are **not** additive across logins, so pass the full set every time.
- Token cache: MSAL stores tokens in a DPAPI-encrypted file under `%LOCALAPPDATA%\.IdentityService\` (Windows). `mgc logout` clears it.

## Discovering IDs

```powershell
# Teams I belong to
mgc users joined-teams list --user-id me `
  --select id,displayName,description --top 50

# Channels in a team
mgc teams channels list --team-id $teamId `
  --select id,displayName,membershipType

# Chats (1:1 / group)
mgc users chats list --user-id me --top 50 `
  --filter "chatType eq 'oneOnOne' or chatType eq 'group'" `
  --select id,topic,chatType,lastUpdatedDateTime
```

Useful filters for `mgc chats list`:

- `--filter "chatType eq 'oneOnOne'"` — DMs only
- `--filter "topic eq 'Project X'"` — by group chat name
- `--orderby "lastUpdatedDateTime desc"` to surface active chats first

## Channel messages

### List recent channel messages

```powershell
mgc teams channels messages list `
  --team-id $teamId --channel-id $channelId `
  --top 25 `
  --select id,from,body,createdDateTime,importance
```

Note: this returns **top-level messages only** — replies live under each message's `replies` navigation property. To page through everything (top-level + replies) use the delta endpoint:

```powershell
mgc teams channels messages delta --team-id $teamId --channel-id $channelId --top 50
```

### Post a channel message (destructive — see Safety Rules)

```powershell
$msg = @'
{
  "body": {
    "contentType": "html",
    "content": "<p>Heads up: deployment window starts at <b>3 PM PT</b>.</p>"
  }
}
'@
$msg | Out-File -Encoding utf8 msg.json

mgc teams channels messages create `
  --team-id $teamId --channel-id $channelId --body "@msg.json"
```

Set `contentType: "text"` for plain text. `importance: "high"` and `"urgent"` are valid for prioritized posts.

### Reply to a channel message

```powershell
mgc teams channels messages replies create `
  --team-id $teamId --channel-id $channelId --chat-message-id $parentMessageId `
  --body "@msg.json"
```

### @-mention a user in a channel

Mentions need a parallel `mentions` array and `<at id="0">…</at>` markers in the body:

```powershell
$mention = @'
{
  "body": {
    "contentType": "html",
    "content": "<p><at id=\"0\">Alice</at> can you take this?</p>"
  },
  "mentions": [
    {
      "id": 0,
      "mentionText": "Alice",
      "mentioned": {
        "user": {
          "displayName": "Alice Example",
          "id": "AAD-OBJECT-ID-OF-ALICE",
          "userIdentityType": "aadUser"
        }
      }
    }
  ]
}
'@
$mention | Out-File -Encoding utf8 mention.json
mgc teams channels messages create --team-id $teamId --channel-id $channelId --body "@mention.json"
```

Resolve the user's `id` via `mgc users list --filter "startswith(displayName,'Alice')"`.

## Chat messages (1:1 and group)

### List chats I'm in

```powershell
mgc users chats list --user-id me --top 50 --select id,topic,chatType,members
```

### Read messages in a chat

```powershell
mgc users chats messages list --user-id me --chat-id $chatId `
  --top 25 --orderby "createdDateTime desc" `
  --select from,body,createdDateTime,messageType
```

### Send a chat message (destructive)

```powershell
'{ "body": { "content": "Quick question — are you free at 3?" } }' |
  Out-File -Encoding utf8 dm.json
mgc users chats messages create --user-id me --chat-id $chatId --body "@dm.json"
```

To start a **new** 1:1 chat (instead of reusing an existing one), POST to `mgc users chats create --user-id me` with `chatType: "oneOnOne"` and both members; then post a message into the returned chat id. This requires the `Chat.Create` scope.

## Adaptive Cards

Adaptive Cards are sent as **attachments** with `contentType` `application/vnd.microsoft.card.adaptive`. The body's `content` references the card by attachment id `<attachment id="…"></attachment>`:

```powershell
$card = @'
{
  "body": {
    "contentType": "html",
    "content": "<attachment id=\"1\"></attachment>"
  },
  "attachments": [
    {
      "id": "1",
      "contentType": "application/vnd.microsoft.card.adaptive",
      "contentUrl": null,
      "content": "{\"$schema\":\"http://adaptivecards.io/schemas/adaptive-card.json\",\"type\":\"AdaptiveCard\",\"version\":\"1.4\",\"body\":[{\"type\":\"TextBlock\",\"text\":\"Build #4242 succeeded\",\"weight\":\"Bolder\",\"size\":\"Medium\"},{\"type\":\"TextBlock\",\"text\":\"Duration: 4m 12s\",\"isSubtle\":true}],\"actions\":[{\"type\":\"Action.OpenUrl\",\"title\":\"View build\",\"url\":\"https://example.com/builds/4242\"}]}",
      "name": null,
      "thumbnailUrl": null
    }
  ]
}
'@
$card | Out-File -Encoding utf8 card.json
mgc teams channels messages create --team-id $teamId --channel-id $channelId --body "@card.json"
```

Key points:

- `attachments[].content` is a **stringified JSON** (note the escaped quotes) — Graph rejects nested objects here.
- Author cards visually at <https://adaptivecards.io/designer/>, then JSON-stringify for embedding.
- The same payload shape works on `mgc users chats messages create --user-id me`.
- Author cards visually at <https://adaptivecards.io/designer/>, then JSON-stringify for embedding.

## Activity feed

```powershell
mgc users teamwork sent-activity-notifications list --user-id me --top 25
mgc users teamwork associated-teams list --user-id me --top 25
```

Activity-feed *sending* (`sendActivityNotification`) requires app-only permissions; out of scope for interactive `mgc login`.

## Safety Rules

1. **Posting is immediate and irreversible** for channel messages. There is no "edit window guarantee" — even though Teams shows an edit affordance, Graph posts cannot be silently un-posted. **Echo the channel name + first 200 chars** of the message back to the user and ask for confirmation before sending.
2. **Confirm the team AND channel name** before posting — channel ids look identical and `19:abc…@thread.tacv2` is easy to mix up. Resolve and show `team.displayName / channel.displayName`.
3. **Don't post to `general` channels** of large orgs without explicit, repeated confirmation — those channels often have hundreds of muted recipients who will still get notifications.
4. **@-mentions notify in real time.** Confirm the resolved AAD user id matches the human the user named.
5. **No mass DMs.** If a script is about to send to >3 distinct chat ids, pause and require explicit user approval per chat.
6. **Card payloads are sanitized server-side** but not the body html — avoid embedding raw user input as HTML without escaping.
7. **Throttling.** Graph limits Teams writes to ~30/sec per app; honor `Retry-After`.

## Notes

- **Federation.** Cross-tenant channel posting requires the channel be a **shared channel** and the user's tenant be federated.
- **Bots vs delegated.** Interactive `mgc login` uses *delegated* permissions — you post as the user. To post **as a bot**, register an app and use client-credentials flow; bots also unlock proactive messaging and richer adaptive-card flows.
- **Attachments on chat messages** (files) use the `chatMessageHostedContent` model — separate upload step required.
- **Reactions / replies / edits** all live under `mgc teams channels messages` and `mgc chats messages` — explore subcommands with `-h`.
- **Personal Microsoft accounts** have no Teams; this skill is work/school only.
