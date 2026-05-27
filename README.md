# n8n YouTube Comment Bot

An end-to-end automation system integrating LINE Messaging API, YouTube Data API v3, and Google Gemini AI via n8n workflow engine. Manage YouTube comments hands-free through conversational LINE configuration, automated keyword-based replies, and real-time AI-generated notifications.

> Note: The workflow JSON files use Chinese node names (e.g. `讀取規則`, `留言過濾`) that match the original development environment. The Code expressions reference these exact Chinese names. If you rename nodes after import, update the references accordingly.

---

## Features

- Natural Language Setup via LINE: Configure monitoring rules through conversational LINE messages. Gemini AI parses unstructured input into structured JSON and writes directly to Google Sheets. Supports single and batch entries.

- Scheduled Auto-Reply: Polls latest YouTube comments every 3 minutes via YouTube Data API v3. Filters out already-processed comments and the channel owner's own replies. Keyword-matched comments receive automated replies on YouTube.

- AI-Generated Real-Time Notifications: Gemini dynamically generates Traditional Chinese LINE notifications categorized as Auto-Replied or Pending Review, helping creators stay updated without checking YouTube manually.

- Privacy-First Self-Hosted: Runs entirely on your local machine via Docker. No third-party cloud workflow services. All credentials and data remain on your device.

---

## Architecture

```
WORKFLOW 1: YT_reply_settings (Rule Setup via LINE)

[LINE User] -> [LINE API] -> [ngrok] -> [n8n Webhook]
                                            |
                                            v
                                    [AI Agent + Gemini]
                                            |
                                            v
                                    [Code: JSON Parse]
                                            |
                                            v
                                  [Google Sheets Append]
                                            |
                                            v
                                [HTTP Request -> LINE Reply]


WORKFLOW 2: YT_auto_reply (Scheduled Comment Processing)

[Schedule Trigger: every 3min]
         |
         v
[Read Sheets Rules]
         |
         v
[Loop Over Items]
         |
         v
[GET YouTube Comments via API]
         |
         v
[Code: Filter + Decide Action (reply/notify/skip)]
         |
         v
[If: action != skip?]
  |- True: [If: action = reply?]
  |        |- True: [YouTube Reply API]
  |        |- False: skip to update
  |              |
  |              v
  |    [Update Sheets: append commentId to F2]
  |              |
  |              v
  |    [AI_Notify Agent + Gemini]
  |              |
  |              v
  |    [LINE Push API]
  |              |
  |              v
  |- False: [Back to Loop]
```

---

## Tech Stack

| Layer | Tools |
|---|---|
| Workflow Engine | n8n (self-hosted) |
| Container | Docker, docker-compose |
| Tunneling | ngrok |
| AI | Google Gemini API (gemini-3-flash-preview) |
| APIs | LINE Messaging API, YouTube Data API v3, Google Sheets API |
| Auth | OAuth 2.0, Bearer Token, Header Auth |
| Database | Google Sheets (rule storage), n8n SQLite (workflow state) |

---

## Prerequisites

- macOS 12 Monterey+ (Apple Silicon or Intel) — Windows and Linux also supported
- Docker Desktop installed and running
- ngrok account (free plan)
- Google account (for YouTube, Sheets, Gemini)
- LINE account (for Messaging API)

---

## Setup Guide

### Phase 1: Environment Setup

#### 1.1 Install Docker Desktop

Download from docker.com and start the daemon.

```bash
docker --version
docker compose version
```

#### 1.2 Get ngrok Token and Domain

1. Sign up at ngrok.com
2. Copy your Authtoken from dashboard
3. Go to Domains -> New Domain -> reserve a free static domain (e.g. your-name.ngrok-free.dev)

#### 1.3 Create docker-compose.yml

```bash
mkdir -p ~/n8n-local
cd ~/n8n-local
nano docker-compose.yml
```

Paste the following (replace YOUR_NGROK_DOMAIN and YOUR_NGROK_TOKEN):

```yaml
services:
  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - N8N_HOST=0.0.0.0
      - N8N_PORT=5678
      - N8N_PROTOCOL=http
      - WEBHOOK_URL=https://YOUR_NGROK_DOMAIN.ngrok-free.dev
      - N8N_EDITOR_BASE_URL=http://localhost:5678
      - GENERIC_TIMEZONE=Asia/Taipei
      - N8N_SECURE_COOKIE=false
    volumes:
      - n8n_data:/home/node/.n8n

  ngrok:
    image: ngrok/ngrok:latest
    container_name: ngrok
    restart: unless-stopped
    command: http --domain=YOUR_NGROK_DOMAIN.ngrok-free.dev n8n:5678
    environment:
      - NGROK_AUTHTOKEN=YOUR_NGROK_TOKEN
    ports:
      - "4040:4040"
    depends_on:
      - n8n

volumes:
  n8n_data:
```

#### 1.4 Launch

```bash
docker compose up -d
```

Open http://localhost:5678 to set up your n8n admin account.

---

### Phase 2: API Credentials

#### 2.1 Google Cloud Console

1. Create project at console.cloud.google.com
2. Enable APIs: Google Sheets API, YouTube Data API v3, Google Drive API
3. Create OAuth consent screen (External) and add yourself as test user
4. Create OAuth 2.0 Client ID (Web Application) with redirect URIs:
   ```
   http://localhost:5678/rest/oauth2-credential/callback
   https://YOUR_NGROK_DOMAIN.ngrok-free.dev/rest/oauth2-credential/callback
   ```
5. Save Client ID and Client Secret

#### 2.2 Gemini API Key

1. Visit aistudio.google.com
2. Get API key (recommend creating a new dedicated project)
3. Enable billing if you encounter quota errors (free trial includes $300 credit)

#### 2.3 LINE Messaging API

1. Visit developers.line.biz
2. Create Provider -> Create LINE Official Account -> Enable Messaging API
3. Get Channel Access Token (long-lived)
4. Get Your User ID from Basic settings
5. Disable auto-reply messages in LINE Official Account Manager
6. Add the bot as friend via QR code

---

### Phase 3: n8n Credentials

Navigate to http://localhost:5678/home/credentials and create:

| Credential Name | Type | Config |
|---|---|---|
| Google Sheets account | Google Sheets OAuth2 API | Client ID + Secret -> Sign in |
| Google OAuth2 account | Google OAuth2 API | Same Client ID + Secret + Scope: https://www.googleapis.com/auth/youtube.force-ssl -> Sign in |
| Google Gemini(PaLM) Api account | Google Gemini(PaLM) API | API Key |
| Header Auth account | Header Auth | Name: Authorization, Value: Bearer YOUR_LINE_CHANNEL_TOKEN |

---

### Phase 4: Google Sheets

Create a new spreadsheet named `YT_Auto_Reply_Comments` (or any name you prefer; just match it in the Sheets node config) with this header row (A1:F1):

| youtube_url | ID | keyword | reply | check_time | replied_ids |
|---|---|---|---|---|---|

> Note: The workflow JSON uses Chinese column names (`關鍵字`, `回應`, `檢查時間`, `已回覆留言ID`). If you use English column names, rename the references in the Code nodes and Sheets mappings accordingly.

---

### Phase 5: Import Workflows

In n8n, click the menu icon and select Import from File. Import the two JSON files:

- workflows/YT_reply_settings.json (Workflow 1)
- workflows/YT_auto_reply.json (Workflow 2)

After import, manually:
1. Re-link credentials in each node (n8n does not auto-bind credentials across imports)
2. Update the ownChannelId in the Code node of Workflow 2 to your YouTube channel ID
3. Update YOUR_LINE_USER_ID in the LINE Push node of Workflow 2 to your LINE User ID

---

## Workflow 1: YT_reply_settings (5 nodes + 1 sub-node)

### Node 1: Webhook

| Field | Value |
|---|---|
| HTTP Method | POST |
| Path | (auto-generated UUID) |
| Respond | Immediately |

Save the Production URL — needed for LINE Webhook setup later.

### Node 2: Google Gemini Chat Model1 (Sub-node)

| Field | Value |
|---|---|
| Credential | Google Gemini(PaLM) Api account |
| Model | models/gemini-3-flash-preview |
| Sampling Temperature | 0 |

### Node 3: AI Agent

| Field | Value |
|---|---|
| Source for Prompt | Define below |
| Prompt (User Message) | `={{ $json.body.events[0].message.text }}` |

System Message:

```
You are a strict data formatter. Your task is to parse user messages into JSON.

Absolute Rules - Never Violate
1. Output JSON only. No explanation text, emoji, or preamble.
2. Never wrap with markdown code fences.
3. No prefix or suffix.
4. For multiple entries: each on its own line, separated by \n.
5. For single entry: output only one line of JSON.

Format
{"url":"full_url","id":"video_id","key":"keyword","reply":"reply_text"}

Field Rules
- id: must be exactly 11 chars (A-Z, a-z, 0-9, -, _)
  Examples:
    https://youtu.be/AcO7g7tS4Ew -> AcO7g7tS4Ew
    https://youtu.be/AcO7g7tS4Ew?si=xxx -> AcO7g7tS4Ew (ignore ?si=)
    https://www.youtube.com/watch?v=AcO7g7tS4Ew -> AcO7g7tS4Ew
- No leading/trailing whitespace on any field
- Strings must be quoted

Output ONLY the JSON. Nothing else.
```

Connect Gemini Chat Model1 to AI Agent's Chat Model slot.

### Node 4: Code in JavaScript

| Field | Value |
|---|---|
| Mode | Run Once for All Items |
| Language | JavaScript |

Code:

```javascript
const raw = $input.item.json.output.replace(/```json|```/g, '').trim();

const lines = raw.split('\n').filter(line => line.trim());
const results = [];

for (const line of lines) {
  try {
    const data = JSON.parse(line.trim());
    
    if (!data.url || !data.id || !data.key || !data.reply) {
      console.log('Missing field, skipping:', line);
      continue;
    }
    
    if (!/^[A-Za-z0-9_-]{11}$/.test(String(data.id).trim())) {
      console.log('Invalid id format, skipping:', data.id);
      continue;
    }
    
    results.push({
      json: {
        url: String(data.url).trim(),
        id: String(data.id).trim(),
        key: String(data.key).trim(),
        reply: String(data.reply).trim()
      }
    });
  } catch (e) {
    console.log('JSON parse failed, skipping:', line, e.message);
  }
}

if (results.length === 0) {
  throw new Error('AI returned no valid data');
}

return results;
```

### Node 5: Google Sheets (Append row in sheet)

| Field | Value |
|---|---|
| Credential | Google Sheets account |
| Resource | Sheet Within Document |
| Operation | Append Row |
| Document | YT_Auto_Reply_Comments |
| Sheet | Sheet1 |
| Mapping Column Mode | Map Each Column Manually |

Column mappings:

| Column | Value |
|---|---|
| youtube_url | `{{ $json.url }}` |
| ID | `{{ $json.id }}` |
| keyword | `{{ $json.key }}` |
| reply | `{{ $json.reply }}` |

### Node 6: HTTP Request (LINE Push)

| Field | Value |
|---|---|
| Method | POST |
| URL | https://api.line.me/v2/bot/message/push |
| Authentication | Generic Credential Type |
| Generic Auth Type | Header Auth |
| Credential | Header Auth account |
| Send Headers | on |
| Specify Headers | Using Fields Below |
| Header Name / Value | Content-Type / application/json |
| Send Body | on |
| Body Content Type | JSON |
| Specify Body | Using JSON |
| Settings -> Execute Once | on |

JSON Body:

```
{{ JSON.stringify({
  to: $('Webhook').item.json.body.events[0].source.userId,
  messages: [{
    type: 'text',
    text: $('Code in JavaScript').all().length === 1
      ? 'Setup complete!\n\nVideo: ' + $('Code in JavaScript').first().json.url
        + '\nKeyword: ' + $('Code in JavaScript').first().json.key
        + '\nAuto-reply: ' + $('Code in JavaScript').first().json.reply
      : 'Setup complete! Added ' + $('Code in JavaScript').all().length + ' rules:\n\n' +
        $('Code in JavaScript').all().map((item, i) =>
          (i+1) + '. Video: ' + item.json.url +
          '\n   Keyword: ' + item.json.key +
          '\n   Reply: ' + item.json.reply
        ).join('\n\n')
  }]
}) }}
```

---

## Workflow 2: YT_auto_reply (12 nodes)

### Node 1: Schedule Trigger

| Field | Value |
|---|---|
| Trigger Interval | Minutes |
| Minutes Between Triggers | 3 |

### Node 2: Read Rules (Google Sheets - Get Rows)

> Node name in JSON: `讀取規則`

| Field | Value |
|---|---|
| Credential | Google Sheets account |
| Operation | Get Row(s) |
| Document | YT_Auto_Reply_Comments |
| Sheet | Sheet1 |

### Node 3: Loop Over Items

| Field | Value |
|---|---|
| Batch Size | 1 |

### Node 4: GET YouTube Comments (HTTP Request)

> Node name in JSON: `GET YouTube留言`

| Field | Value |
|---|---|
| Method | GET |
| URL | https://www.googleapis.com/youtube/v3/commentThreads |
| Authentication | Predefined Credential Type |
| Credential Type | Google OAuth2 API |
| Credential | Google OAuth2 account |
| Send Query Parameters | on |

Query Parameters:

| Name | Value |
|---|---|
| part | snippet |
| videoId | `{{ $('Loop Over Items').item.json['ID'] }}` |
| maxResults | 20 |
| order | time |

### Node 5: Comment Filter (Code in JavaScript)

> Node name in JSON: `留言過濾`

Code (Replace ownChannelId with your YouTube channel ID):

```javascript
const loopRow = $('Loop Over Items').item.json;
const ytItems = $input.first().json.items || [];

// Force string conversion + trim to handle numeric or TAB-prefixed values
const keyword    = String(loopRow['關鍵字'] || '').trim();
const replyText  = String(loopRow['回應'] || '').trim();
const existingIds = String(loopRow['已回覆留言ID'] || '').trim();
const ownChannelId = 'UCxxxxxxxxxxxxxxxxxxx'; // Replace with your channel ID

// Aggregate processed IDs across all rows of the same video
const allRows = $('讀取規則').all().map(i => i.json);
const existingList = allRows
  .filter(r => String(r['youtube_url'] || '').trim() === String(loopRow['youtube_url'] || '').trim())
  .flatMap(r => String(r['已回覆留言ID'] || '').trim().split(',').filter(x => x.trim()));

// Filter: exclude already-processed + own channel
const newComments = ytItems.filter(item => {
  const id     = item.snippet.topLevelComment.id;
  const author = item.snippet.topLevelComment.snippet.authorChannelId?.value;
  return !existingList.includes(id) && author !== ownChannelId;
});

// Categorize
const toReply  = newComments.filter(c =>
  c.snippet.topLevelComment.snippet.textDisplay.includes(keyword)
);
const toNotify = newComments.filter(c =>
  !c.snippet.topLevelComment.snippet.textDisplay.includes(keyword)
);

// Pick ONE per execution (priority: reply > notify)
const target = toReply[0] || toNotify[0] || null;
if (!target) return [{ json: { action: 'skip' } }];

const action = toReply.includes(target) ? 'reply' : 'notify';

// Guard: notify only from the row with smallest row_number (avoid duplicate notifications)
if (action === 'notify') {
  const sameVideoMinRow = Math.min(...allRows
    .filter(r => String(r['youtube_url'] || '').trim() === String(loopRow['youtube_url'] || '').trim())
    .map(r => r['row_number']));
  if (loopRow['row_number'] !== sameVideoMinRow) {
    return [{ json: { action: 'skip' } }];
  }
}

const comment = target.snippet.topLevelComment;

return [{ json: {
  action,
  row_number:  loopRow['row_number'],
  commentId:   comment.id,
  text:        comment.snippet.textDisplay,
  author:      comment.snippet.authorDisplayName,
  videoId:     target.snippet.videoId,
  youtube_url: String(loopRow['youtube_url'] || '').trim(),
  replyText,
  existingIds
}}];
```

> Note: The Code references Chinese column names (`關鍵字`, `回應`, `已回覆留言ID`) because the workflow JSON was developed with Chinese Sheets columns. If you use English column names, update these references.

### Node 6: Has Action? (If)

> Node name in JSON: `有動作?`

| Field | Value |
|---|---|
| First Value | `{{ $json.action }}` |
| Operation | is not equal to |
| Second Value | skip |

- True -> Node 7 (Needs Reply?)
- False -> Loop Over Items (back)

### Node 7: Needs Reply? (If)

> Node name in JSON: `需要回覆?`

| Field | Value |
|---|---|
| First Value | `{{ $json.action }}` |
| Operation | is equal to |
| Second Value | reply |

- True -> Node 8 (YouTube Reply)
- False -> Node 9 (Update Reply Records)

### Node 8: YouTube Reply (HTTP Request)

> Node name in JSON: `YouTube回覆`

| Field | Value |
|---|---|
| Method | POST |
| URL | https://www.googleapis.com/youtube/v3/comments |
| Authentication | Predefined Credential Type |
| Credential Type | Google OAuth2 API |
| Credential | Google OAuth2 account |
| Send Query Parameters | on (part: snippet) |
| Send Body | on |
| Body Content Type | JSON |
| Specify Body | Using JSON |

JSON Body:

```
{{ JSON.stringify({
  snippet: {
    parentId: $('留言過濾').item.json.commentId,
    textOriginal: $('留言過濾').item.json.replyText
  }
}) }}
```

### Node 9: Update Reply Records (Google Sheets - Update Row)

> Node name in JSON: `更新已回覆紀錄`

| Field | Value |
|---|---|
| Credential | Google Sheets account |
| Operation | Update Row |
| Document | YT_Auto_Reply_Comments |
| Sheet | Sheet1 |
| Column to match on | row_number |

Column mappings:

| Column | Value |
|---|---|
| row_number | `{{ $('留言過濾').item.json['row_number'] }}` |
| youtube_url | `{{ $('留言過濾').item.json['youtube_url'] }}` |
| check_time | `{{ $now.toFormat('yyyy-MM-dd HH:mm:ss') }}` |
| replied_ids | `{{ ($('留言過濾').item.json['existingIds'] ? $('留言過濾').item.json['existingIds'] + ',' : '') + $('留言過濾').item.json['commentId'] }}` |

### Node 10: Google Gemini Chat Model (Sub-node)

| Field | Value |
|---|---|
| Credential | Google Gemini(PaLM) Api account |
| Model | models/gemini-3-flash-preview |
| Sampling Temperature | 0.3 |

### Node 11: AI_Notify (AI Agent)

| Field | Value |
|---|---|
| Source for Prompt | Define below |

Prompt:

```
={{ $('留言過濾').item.json.action === 'reply'
  ? 'Auto-replied to @' + $('留言過濾').item.json.author
    + '\'s comment: "' + $('留言過濾').item.json.text + '"'
    + ', reply: ' + $('留言過濾').item.json.replyText
  : 'New comment needs review, from: @' + $('留言過濾').item.json.author
    + ', content: "' + $('留言過濾').item.json.text + '"'
}}
```

System Message:

```
You are a LINE notification assistant for a YouTube channel.

Task
Generate a concise LINE notification in Traditional Chinese based on the provided comment info.

Format Rules - Strict
1. Auto-replied notifications start with a checkmark symbol
2. Pending-review notifications start with a warning symbol
3. Total length under 100 chars
4. Max 4 lines
5. No prefix like "Here is..." or "Notification:"
6. No markdown formatting
7. Output the message directly
```

Connect Gemini Chat Model to AI_Notify's Chat Model slot.

### Node 12: LINE Push (HTTP Request)

> Node name in JSON: `LINE推播`

| Field | Value |
|---|---|
| Method | POST |
| URL | https://api.line.me/v2/bot/message/push |
| Authentication | Generic Credential Type |
| Generic Auth Type | Header Auth |
| Credential | Header Auth account |
| Send Headers | on |
| Header Name / Value | Content-Type / application/json |
| Send Body | on |
| Body Content Type | JSON |
| Specify Body | Using JSON |

JSON Body (Replace YOUR_LINE_USER_ID):

```
{{ JSON.stringify({
  to: 'YOUR_LINE_USER_ID',
  messages: [{
    type: 'text',
    text: $json.output + '\n\nVideo link: ' + $('留言過濾').item.json.youtube_url
  }]
}) }}
```

### Final Connections

- LINE Push output -> back to Loop Over Items input
- Has Action? false -> back to Loop Over Items input
- Delete any auto-generated loop-back from GET YouTube Comments directly to Loop

---

## LINE Webhook Configuration

1. Open LINE Developers Console
2. Navigate to your Messaging API channel -> Messaging API tab
3. Set Webhook URL to Workflow 1's Production URL:
   ```
   https://YOUR_NGROK_DOMAIN.ngrok-free.dev/webhook/YOUR_WEBHOOK_UUID
   ```
4. Click Verify (should return Success)
5. Enable Use webhook toggle

---

## Testing

### Test Workflow 1

Send to your LINE bot:

```
https://youtu.be/dQw4w9WgXcQ keyword:subscribe reply:thanks for subscribing
```

Or in Chinese (matches the original development):

```
https://youtu.be/dQw4w9WgXcQ 關鍵字：訂閱 回覆：感謝訂閱
```

Expected:
- LINE bot replies with "Setup complete!"
- New row in Google Sheets

### Test Workflow 2

1. Have a friend (or alternate account) comment "I subscribed" (or "我訂閱了") on your video
2. Manually trigger Workflow 2 or wait 3 minutes
3. Expected:
   - YouTube reply appears under the friend's comment
   - LINE push notification arrives

---

## Troubleshooting

| Issue | Solution |
|---|---|
| videoNotFound 404 | Comments disabled on video / video is Unlisted or Private / "Made for Kids" set to Yes / YouTube Short (API limited) |
| Gemini 429 quota: 0 | Enable Cloud Billing on the project linked to your Gemini key |
| Invalid reply token | LINE Reply tokens expire in 30s — use Push API instead |
| JSON parse failed at position N | AI returned multiple JSONs; updated Code already handles this with split('\n') |
| .trim is not a function | Sheets returned a number; updated Code wraps values with String() |
| Loop only processes 1 row | Confirm Read Rules node returns all items (no "Return only First Matching Row" option) |
| HTTP Request runs N times | Enable Execute Once in Settings tab |
| LINE comments disappear after refresh | YouTube spam filter — diversify reply text, avoid template responses |

---

## Repository Structure

```
n8n-YouTube-Comment-Bot/
├── README.md
├── docker-compose.example.yml
├── workflows/
│   ├── YT_reply_settings.json
│   └── YT_auto_reply.json
├── docs/
│   ├── architecture1.png
│   └── architecture2.png
└── .gitignore
```

---

## Acknowledgments

- n8n — workflow automation engine
- Google Gemini — LLM
- LINE Messaging API
- YouTube Data API v3
