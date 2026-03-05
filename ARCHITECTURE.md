# OpenClaw System Architecture

## High-Level Overview

```
+------------------------------------------------------------------+
|                        User / External                           |
|                                                                  |
|  CLI (TUI)    Web UI    WhatsApp   Telegram   Discord   Slack    |
|     |           |          |          |          |         |      |
+-----|-----------|----------|----------|----------|---------|------+
      |           |          |          |          |         |
      v           v          v          v          v         v
+------------------------------------------------------------------+
|                      Entry Points Layer                          |
|                                                                  |
|  src/cli/        src/web/       src/channels/                    |
|  (CLI + TUI)     (Web UI)       (Channel Adapters)               |
|                                                                  |
|  src/whatsapp/  src/telegram/  src/discord/  src/slack/          |
|  src/imessage/  src/line/      src/signal/                       |
+------------------------------|-----------------------------------+
                               |
                               v
+------------------------------------------------------------------+
|                       Gateway Layer                              |
|                                                                  |
|  src/gateway/                                                    |
|  +----------------------------------------------------------+   |
|  | server-http.ts          HTTP Server & Request Router      |   |
|  |  - POST /hooks/*        Webhook endpoints                 |   |
|  |  - Auth (Bearer / x-openclaw-token)                       |   |
|  +----------------------------------------------------------+   |
|  | hooks.ts                Config resolution, auth, parsing  |   |
|  | hooks-mapping.ts        Mapping match, templates,         |   |
|  |                         transform modules                 |   |
|  | server/hooks.ts         Dispatch: wake / agent handlers   |   |
|  +----------------------------------------------------------+   |
+------------------------------|-----------------------------------+
                               |
            +------------------+------------------+
            |                                     |
            v                                     v
+------------------------+          +---------------------------+
|     Wake Action        |          |      Agent Action         |
|                        |          |                           |
| enqueueSystemEvent()   |          | runCronIsolatedAgentTurn()|
| requestHeartbeatNow()  |          | (isolated session)        |
|                        |          | result -> main session    |
+-----------|------------+          +-------------|-------------+
            |                                     |
            +------------------+------------------+
                               |
                               v
+------------------------------------------------------------------+
|                        Agent Core                                |
|                                                                  |
|  src/agents/                                                     |
|  +----------------------------------------------------------+   |
|  | bootstrap-hooks.ts     Agent init & bootstrap phase       |   |
|  | (agent turns)          Message processing & tool use      |   |
|  +----------------------------------------------------------+   |
|                                                                  |
|  src/hooks/                                                      |
|  +----------------------------------------------------------+   |
|  | internal-hooks.ts      Event hook system                  |   |
|  |  - command / session / agent / gateway events             |   |
|  |  - registerInternalHook() / triggerInternalHook()         |   |
|  +----------------------------------------------------------+   |
+------------------------------|-----------------------------------+
                               |
            +------------------+------------------+
            |                  |                  |
            v                  v                  v
+----------------+  +------------------+  +------------------+
| Sessions       |  | Providers        |  | Tools / Plugins  |
|                |  |                  |  |                  |
| src/sessions/  |  | src/providers/   |  | src/plugins/     |
| Session keys   |  | LLM routing      |  | src/plugin-sdk/  |
| State mgmt     |  | Model selection  |  | src/browser/     |
| History        |  | API calls        |  | src/terminal/    |
+----------------+  +------------------+  +------------------+

+------------------------------------------------------------------+
|                     Supporting Systems                            |
|                                                                  |
|  src/config/          Configuration & validation (Zod schemas)   |
|  src/cron/            Scheduled jobs & periodic tasks            |
|  src/memory/          Persistent memory & context                |
|  src/security/        Auth, sandboxing, content safety           |
|  src/routing/         Request routing logic                      |
|  src/media/           Media handling & understanding             |
|  src/logging/         Structured logging                         |
|  src/utils/           Shared utilities                           |
+------------------------------------------------------------------+
```

## Hook Mapping Request Flow

```
                    External Service (e.g. Gmail Pub/Sub)
                                |
                                | POST /hooks/gmail
                                | Authorization: Bearer <token>
                                | { "messages": [{ "from": "...", "subject": "..." }] }
                                v
                   +------------------------+
                   |   server-http.ts       |
                   |                        |
                   |  1. Check hooks.enabled|
                   |  2. Match path prefix  |
                   |  3. Validate auth token|
                   |  4. Parse JSON body    |
                   |  5. Route by subPath   |
                   +-----------|------------+
                               |
                    subPath = "gmail"
                               |
                               v
                   +------------------------+
                   |   hooks-mapping.ts     |
                   |                        |
                   |  1. Iterate mappings   |
                   |  2. Match path/source  |
                   |  3. Render templates:  |
                   |     {{messages[0].from}|
                   |  4. Load transform     |
                   |     module (optional)  |
                   |  5. Merge action       |
                   +-----------|------------+
                               |
                   action = "wake" | "agent"
                               |
              +----------------+----------------+
              |                                 |
              v                                 v
  +-----------------------+       +---------------------------+
  |   Wake Handler        |       |   Agent Handler           |
  |                       |       |                           |
  | enqueueSystemEvent()  |       | Create CronJob            |
  | heartbeat if "now"    |       | sessionTarget: "isolated" |
  |                       |       | Run agent turn async      |
  | Response: 200 OK      |       | Post summary to main      |
  +-----------------------+       |                           |
                                  | Response: 202 Accepted    |
                                  +---------------------------+
```

## Configuration Hierarchy

```
+-----------------------------------------------+
|              HooksConfig                       |
|                                                |
|  enabled: boolean                              |
|  path: string ("/hooks")                       |
|  token: string (required)                      |
|  maxBodyBytes: number (256KB)                  |
|                                                |
|  presets: ["gmail"]  ----+                     |
|                          |  merged             |
|  mappings: [  <----------+                     |
|    {                                           |
|      match: { path, source }                   |
|      action: "wake" | "agent"                  |
|      wakeMode: "now" | "next-heartbeat"        |
|      sessionKey: "hook:gmail:{{..}}"           |
|      messageTemplate: "From {{..}}"            |
|      transform: { module, export }             |
|      channel: "last"|"whatsapp"|"telegram"|... |
|      model / thinking / timeoutSeconds         |
|      deliver / allowUnsafeExternalContent      |
|    }                                           |
|  ]                                             |
|                                                |
|  gmail: { model, thinking }                    |
|  internal: { ... }                             |
+-----------------------------------------------+
```

## Internal Event Hook System

```
  registerInternalHook("command:new", handler)
  registerInternalHook("session:start", handler)
  registerInternalHook("agent:bootstrap", handler)

                    Event Fired
                        |
                        v
  +------------------------------------------+
  |  triggerInternalHook(event)               |
  |                                          |
  |  event = {                               |
  |    type: "command"|"session"|"agent"|     |
  |          "gateway"                        |
  |    action: "new"|"reset"|"bootstrap"|... |
  |    sessionKey, context, timestamp         |
  |  }                                       |
  |                                          |
  |  1. Run handlers for event.type          |
  |  2. Run handlers for type:action         |
  |  3. Errors caught, don't block others    |
  +------------------------------------------+
```
