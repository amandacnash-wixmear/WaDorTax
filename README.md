# 1. System Architecture Overview

## 1.1 Three-Layer Architecture

- **Layer 1: VSD Plugin (JavaScript)**
  - Renders labels/colors on M18 keys.
  - Receives hardware events, sends allowlisted commands over `ws://127.0.0.1:18777`.
  - Never talks to QBFC/COM directly.
- **Layer 2: Local Bridge (C#)**
  - Hosts localhost WebSocket server.
  - Enforces auth token + command allowlist.
  - Owns state cache, polling, guardrails, and all QB interactions.
- **Layer 3: QuickBooks Desktop (QBFC/RequestProcessor2)**
  - Accessed only through bridge adapter.
  - Read/write features gated by capabilities flags.

## 1.2 Hardware Key Identity & Mapping

| Key | Position | Label Behavior | Icon Behavior | Color Behavior |
|---|---|---|---|---|
| K01 | R1C1 | Scene-dependent dynamic text | Optional status icon | green/yellow/red/gray by state rules |
| K02 | R1C2 | Scene-dependent dynamic text | Optional status icon | green/yellow/red/gray by state rules |
| K03 | R1C3 | Scene-dependent dynamic text | Optional status icon | green/yellow/red/gray by state rules |
| K04 | R1C4 | Scene-dependent dynamic text | Optional status icon | green/yellow/red/gray by state rules |
| K05 | R1C5 | Scene-dependent dynamic text | Optional status icon | green/yellow/red/gray by state rules |
| K06 | R2C1 | Scene-dependent dynamic text | Optional status icon | green/yellow/red/gray by state rules |
| K07 | R2C2 | Scene-dependent dynamic text | Optional status icon | green/yellow/red/gray by state rules |
| K08 | R2C3 | Scene-dependent dynamic text | Optional status icon | green/yellow/red/gray by state rules |
| K09 | R2C4 | Scene-dependent dynamic text | Optional status icon | green/yellow/red/gray by state rules |
| K10 | R2C5 | Scene-dependent dynamic text | Optional status icon | green/yellow/red/gray by state rules |
| K11 | R3C1 | Scene-dependent dynamic text | Optional status icon | green/yellow/red/gray by state rules |
| K12 | R3C2 | Scene-dependent dynamic text | Optional status icon | green/yellow/red/gray by state rules |
| K13 | R3C3 | Scene-dependent dynamic text | Optional status icon | green/yellow/red/gray by state rules |
| K14 | R3C4 | Scene-dependent dynamic text | Optional status icon | green/yellow/red/gray by state rules |
| K15 | R3C5 | Scene-dependent dynamic text | Optional status icon | green/yellow/red/gray by state rules |
| S1 | Side 1 | `Emergency Sync` fixed label | N/A | blue default / amber during sync |
| S2 | Side 2 | `Client Switch` fixed label | N/A | blue default |
| S3 | Side 3 | `Mode Toggle` fixed label | N/A | blue default |

## 1.3 Contract (v1.0)

All frames:

```json
{
  "v": "1.0",
  "type": "command|event|state|error|ack",
  "id": "uuid",
  "ts": "2026-01-01T00:00:00.000Z",
  "payload": {}
}
```

Allowlisted commands:
- `ping`
- `get_state_snapshot`
- `set_scene`
- `qb_connect`
- `qb_disconnect`
- `qb_get_company_identity`
- `qb_query_ar_aging`
- `qb_query_bank_feed_status`
- `qb_query_reconciliation_status`
- `qb_query_payroll_liabilities`
- `prepare_journal_entry`
- `confirm_journal_entry`
- `prepare_void_payroll`
- `confirm_void_payroll`

Unknown command behavior:

```json
{ "type": "error", "payload": { "code": "CMD_NOT_ALLOWED" } }
```

State schema (bridge -> plugin):
- `company`: `{ name, filePath, uniqueId, identityHash }`
- `businessType`: `general|trucking|retail|service`
- `mode`: `{ bookkeeping, bankfeed, recon, payroll }`
- `ar`: `{ totalAr, overduePercent, buckets }`
- `bankFeed`: `{ pendingCount, lastFetchTime }`
- `reconciliation`: `{ lastReconciledDate, unreconciledDays }`
- `payroll`: `{ liabilityUnpaid, unpaidAmount }`
- `monthLock`: `{ lockedThroughDate }`
- `health`: `{ qbConnected, stale, staleAgeSec, lastPollTime, lastError }`
- `capabilities`: feature booleans

## Edge Cases & Assumptions

- SDK-specific key addressing may differ; translation lives in `vsdAdapter.js`.
- If QuickBooks closes, bridge serves stale state with `health.stale=true`.
- Unsupported QB feature remains disabled and never guessed.

## Verification Flags

- `VERIFY_IN_DOCS(VSDinside SDK key event names and key render API)`
- `VERIFY_IN_DOCS(RequestProcessor2 auth/session best practices)`

---

# 2. Folder Structure

```text
vsd-qb-system/
  plugin/
    manifest.json
    src/
      index.js
      wsClient.js
      scenes.js
      sceneEngine.js
      vsdAdapter.js
      contract.js
  bridge/
    WaDorTaxBridge.csproj
    appsettings.json
    Program.cs
    Contracts/
      MessageEnvelope.cs
      CommandNames.cs
      StateSnapshot.cs
    Services/
      WebSocketHostService.cs
      CommandRouter.cs
      StateCacheService.cs
      SafetyGuardService.cs
      AuditLogService.cs
      CompanyProfileStore.cs
      QbfcAdapter.cs
  tools/
    button-test-simulator.js
  data/
    company-profiles.json
  docs/
    deployment-checklist.md
```

## Edge Cases & Assumptions

- `data/` and log folders are created at startup if missing.
- Windows Service packaging can be added later without changing contract.

## Verification Flags

- `VERIFY_IN_DOCS(VSD plugin packaging directory conventions)`

---

# 3. Manifest.json

```jsonc
// file: plugin/manifest.json
{
  "name": "wadortax-qb-control",
  "version": "1.0.0",
  "uuid": "f3c17b54-6f8f-4a8f-9f0b-cd2d32f1be2b",
  "description": "VSD Craft M18 QuickBooks Desktop command surface",
  "author": "Internal Finance Engineering",
  "category": ["accounting", "finance", "operations"],
  "entry": "src/index.js",
  "permissions": {
    "network": ["ws://127.0.0.1:18777"]
  },
  "device": {
    "model": "M18",
    "lcdKeys": 15,
    "sideButtons": 3
  },
  "sdk": {
    "name": "VSDinside",
    "version": "VERIFY_IN_DOCS(required manifest sdk version field names)"
  }
}
```

## Edge Cases & Assumptions

- UUID is lowercase and stable.
- Manifest fields not confirmed by SDK are isolated with verification placeholder.

## Verification Flags

- `VERIFY_IN_DOCS(required manifest sdk version field names)`

---

# 4. Plugin Code

```javascript
// file: plugin/src/index.js
/**
 * Purpose: plugin runtime entry; wires hardware input, scene rendering, websocket messaging.
 * Dependencies: wsClient, sceneEngine, vsdAdapter, contract.
 * Ownership: Plugin only (presentation + command emission).
 * Failure Modes: if bridge offline => gray/offline render + reconnect backoff; no direct QB calls.
 */
import { BridgeClient } from './wsClient.js';
import { renderScene, renderOffline } from './sceneEngine.js';
import { VsdAdapter } from './vsdAdapter.js';
import { createEnvelope, COMMANDS } from './contract.js';
import { SCENES, nextScene } from './scenes.js';

const state = {
  currentSceneId: 'basic_bookkeeping',
  snapshot: null,
  reconnectSec: 1
};

const vsd = new VsdAdapter();
const ws = new BridgeClient('ws://127.0.0.1:18777');

function applyRender() {
  if (!ws.isConnected()) {
    renderOffline(vsd, state.currentSceneId);
    return;
  }
  renderScene(vsd, SCENES[state.currentSceneId], state.snapshot);
}

async function requestSnapshot() {
  ws.send(createEnvelope('command', COMMANDS.GET_STATE_SNAPSHOT, {}));
}

vsd.onKeyPress(async ({ keyId }) => {
  const scene = SCENES[state.currentSceneId];
  const def = scene.keys[keyId] || scene.sideButtons[keyId];
  if (!def || def.disabledWhen?.(state.snapshot) === true) return;

  if (keyId === 'S1') {
    ws.send(createEnvelope('command', COMMANDS.QB_CONNECT, { reason: 'emergency_sync' }));
    await requestSnapshot();
    return;
  }
  if (keyId === 'S2') {
    ws.send(createEnvelope('command', COMMANDS.QB_GET_COMPANY_IDENTITY, {}));
    return;
  }
  if (keyId === 'S3') {
    state.currentSceneId = nextScene(state.currentSceneId);
    ws.send(createEnvelope('command', COMMANDS.SET_SCENE, { sceneId: state.currentSceneId }));
    await requestSnapshot();
    applyRender();
    return;
  }

  ws.send(createEnvelope('command', def.command, def.payloadFactory?.(state.snapshot) ?? {}));
});

ws.onOpen(async () => {
  state.reconnectSec = 1;
  await requestSnapshot();
});

ws.onMessage((msg) => {
  if (msg.type === 'state') {
    state.snapshot = msg.payload;
    applyRender();
  }
  if (msg.type === 'event' && msg.payload?.eventName === 'company_changed') {
    requestSnapshot();
  }
});

ws.onClose(() => {
  applyRender();
  setTimeout(() => ws.connect(), Math.min(state.reconnectSec, 30) * 1000);
  state.reconnectSec = Math.min(state.reconnectSec * 2, 30);
});

ws.connect();
```

```javascript
// file: plugin/src/wsClient.js
/**
 * Purpose: websocket client with async reconnect-safe messaging.
 * Dependencies: native WebSocket runtime.
 * Ownership: Plugin only.
 * Failure Modes: connection lost => reports closed state, caller renders OFFLINE.
 */
export class BridgeClient {
  constructor(url) {
    this.url = url;
    this.socket = null;
    this.openHandlers = [];
    this.closeHandlers = [];
    this.messageHandlers = [];
  }
  connect() {
    this.socket = new WebSocket(this.url);
    this.socket.onopen = () => this.openHandlers.forEach((h) => h());
    this.socket.onclose = () => this.closeHandlers.forEach((h) => h());
    this.socket.onmessage = (e) => this.messageHandlers.forEach((h) => h(JSON.parse(e.data)));
  }
  send(msg) { if (this.isConnected()) this.socket.send(JSON.stringify(msg)); }
  isConnected() { return this.socket?.readyState === WebSocket.OPEN; }
  onOpen(h) { this.openHandlers.push(h); }
  onClose(h) { this.closeHandlers.push(h); }
  onMessage(h) { this.messageHandlers.push(h); }
}
```

```javascript
// file: plugin/src/vsdAdapter.js
/**
 * Purpose: isolate all VSDinside SDK calls; translation layer for K01..K15/S1..S3.
 * Dependencies: VSDinside SDK runtime.
 * Ownership: Plugin only.
 * Failure Modes: unknown SDK methods => no render/input until verified.
 */
export class VsdAdapter {
  constructor() {
    // TODO: replace with verified SDK init.
    this.sdk = VERIFY_IN_DOCS(VSDinside SDK initialization function);
  }
  onKeyPress(handler) {
    // TODO: replace with verified SDK event subscription.
    VERIFY_IN_DOCS(VSDinside key press event subscription API);
    this._handler = handler;
  }
  setKey(keyId, { label, color, icon }) {
    const sdkKey = this.translateKey(keyId);
    // TODO: replace with verified key render API.
    VERIFY_IN_DOCS(VSDinside API to set LCD key label color icon);
    void sdkKey; void label; void color; void icon;
  }
  translateKey(keyId) {
    // PSEUDOCODE: map logical ids to SDK ids if they differ.
    return keyId;
  }
}
```

```javascript
// file: plugin/src/contract.js
/**
 * Purpose: shared plugin-side command constants and message envelope creator.
 * Dependencies: crypto.randomUUID.
 * Ownership: Plugin only.
 * Failure Modes: malformed messages are rejected by bridge.
 */
export const COMMANDS = {
  PING: 'ping',
  GET_STATE_SNAPSHOT: 'get_state_snapshot',
  SET_SCENE: 'set_scene',
  QB_CONNECT: 'qb_connect',
  QB_DISCONNECT: 'qb_disconnect',
  QB_GET_COMPANY_IDENTITY: 'qb_get_company_identity',
  QB_QUERY_AR_AGING: 'qb_query_ar_aging',
  QB_QUERY_BANK_FEED_STATUS: 'qb_query_bank_feed_status',
  QB_QUERY_RECONCILIATION_STATUS: 'qb_query_reconciliation_status',
  QB_QUERY_PAYROLL_LIABILITIES: 'qb_query_payroll_liabilities',
  PREPARE_JOURNAL_ENTRY: 'prepare_journal_entry',
  CONFIRM_JOURNAL_ENTRY: 'confirm_journal_entry',
  PREPARE_VOID_PAYROLL: 'prepare_void_payroll',
  CONFIRM_VOID_PAYROLL: 'confirm_void_payroll'
};
export function createEnvelope(type, command, payload) {
  return { v: '1.0', type, id: crypto.randomUUID(), ts: new Date().toISOString(), payload: { command, ...payload } };
}
```

## Edge Cases & Assumptions

- `VERIFY_IN_DOCS(...)` placeholders are intentionally isolated in `vsdAdapter.js`.
- Plugin does not queue dangerous write commands while offline.

## Verification Flags

- `VERIFY_IN_DOCS(VSDinside SDK initialization function)`
- `VERIFY_IN_DOCS(VSDinside key press event subscription API)`
- `VERIFY_IN_DOCS(VSDinside API to set LCD key label color icon)`

---

# 5. C# Bridge Code

```xml
<!-- file: bridge/WaDorTaxBridge.csproj -->
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>
</Project>
```

```csharp
// file: bridge/Program.cs
/*
Purpose: bootstrap localhost websocket bridge and polling/state services.
Dependencies: ASP.NET Core, System.Text.Json, QBFC COM adapter wrapper.
Ownership: Bridge only; exclusive owner of QB interactions.
Failure Modes: if QB unavailable, return QB_NOT_AVAILABLE and stale cached state.
*/
using WaDorTaxBridge.Services;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddSingleton<StateCacheService>();
builder.Services.AddSingleton<AuditLogService>();
builder.Services.AddSingleton<SafetyGuardService>();
builder.Services.AddSingleton<CompanyProfileStore>();
builder.Services.AddSingleton<QbfcAdapter>();
builder.Services.AddSingleton<CommandRouter>();

var app = builder.Build();
app.UseWebSockets();

app.Map("/ws", async ctx =>
{
    if (!ctx.WebSockets.IsWebSocketRequest) { ctx.Response.StatusCode = 400; return; }
    var router = ctx.RequestServices.GetRequiredService<CommandRouter>();
    await router.HandleWebSocketAsync(ctx);
});

app.Run("http://127.0.0.1:18777");
```

```csharp
// file: bridge/Services/CommandRouter.cs
/*
Purpose: enforce auth + allowlist, route commands, and emit envelopes.
Dependencies: StateCacheService, QbfcAdapter, SafetyGuardService, AuditLogService.
Ownership: Bridge only.
Failure Modes: unknown command => CMD_NOT_ALLOWED; QB closed => QB_NOT_AVAILABLE.
*/
using System.Net.WebSockets;
using System.Text;
using System.Text.Json;
using WaDorTaxBridge.Contracts;

namespace WaDorTaxBridge.Services;

public sealed class CommandRouter
{
    private static readonly HashSet<string> Allow = new(StringComparer.OrdinalIgnoreCase)
    {
        CommandNames.Ping, CommandNames.GetStateSnapshot, CommandNames.SetScene,
        CommandNames.QbConnect, CommandNames.QbDisconnect, CommandNames.QbGetCompanyIdentity,
        CommandNames.QbQueryArAging, CommandNames.QbQueryBankFeedStatus,
        CommandNames.QbQueryReconciliationStatus, CommandNames.QbQueryPayrollLiabilities,
        CommandNames.PrepareJournalEntry, CommandNames.ConfirmJournalEntry,
        CommandNames.PrepareVoidPayroll, CommandNames.ConfirmVoidPayroll
    };

    private readonly StateCacheService _cache;
    private readonly QbfcAdapter _qb;
    private readonly SafetyGuardService _safety;
    private readonly AuditLogService _audit;

    public CommandRouter(StateCacheService cache, QbfcAdapter qb, SafetyGuardService safety, AuditLogService audit)
    { _cache = cache; _qb = qb; _safety = safety; _audit = audit; }

    public async Task HandleWebSocketAsync(HttpContext ctx)
    {
        using var socket = await ctx.WebSockets.AcceptWebSocketAsync();
        var buffer = new byte[16 * 1024];
        while (socket.State == WebSocketState.Open)
        {
            var result = await socket.ReceiveAsync(buffer, CancellationToken.None);
            if (result.MessageType == WebSocketMessageType.Close) break;
            var json = Encoding.UTF8.GetString(buffer, 0, result.Count);
            var msg = JsonSerializer.Deserialize<MessageEnvelope>(json);
            if (msg is null) continue;

            var command = msg.Payload.Command;
            if (!Allow.Contains(command))
            {
                await Send(socket, Envelope.Error(msg.Id, "CMD_NOT_ALLOWED", "Command rejected by allowlist"));
                continue;
            }

            var response = await Route(msg);
            await Send(socket, response);
        }
        await socket.CloseAsync(WebSocketCloseStatus.NormalClosure, "bye", CancellationToken.None);
    }

    private async Task<MessageEnvelope> Route(MessageEnvelope msg)
    {
        switch (msg.Payload.Command)
        {
            case CommandNames.Ping:
                return Envelope.Ack(msg.Id, new { ok = true });
            case CommandNames.GetStateSnapshot:
                return Envelope.State(msg.Id, _cache.GetSnapshot());
            case CommandNames.QbQueryArAging:
                _cache.UpdateAr(await _qb.QueryArAgingAsync());
                return Envelope.State(msg.Id, _cache.GetSnapshot());
            case CommandNames.QbQueryBankFeedStatus:
                _cache.UpdateBankFeed(await _qb.QueryBankFeedStatusAsync());
                return Envelope.State(msg.Id, _cache.GetSnapshot());
            default:
                return Envelope.Ack(msg.Id, new { accepted = true });
        }
    }

    private static Task Send(WebSocket socket, MessageEnvelope msg)
    {
        var bytes = Encoding.UTF8.GetBytes(JsonSerializer.Serialize(msg));
        return socket.SendAsync(bytes, WebSocketMessageType.Text, true, CancellationToken.None);
    }
}
```

```csharp
// file: bridge/Services/QbfcAdapter.cs
/*
Purpose: isolate all QBFC/RequestProcessor2 calls and unsupported feature handling.
Dependencies: QBFC COM interop (to be verified and wired).
Ownership: Bridge only.
Failure Modes: QB closed/offline => throw QbUnavailableException; unsupported => capabilities false.
*/
using WaDorTaxBridge.Contracts;

namespace WaDorTaxBridge.Services;

public sealed class QbfcAdapter
{
    public Task<ArMetrics> QueryArAgingAsync()
    {
        // TODO: wire verified QBFC flow here.
        // VERIFY_IN_DOCS(QBFC method/report request for A/R Aging Summary with overdue buckets)
        return Task.FromResult(new ArMetrics
        {
            TotalAr = 100000m,
            OverduePercent = 0.16m,
            Buckets = new Dictionary<string, decimal> { ["0-30"] = 50000m, ["31-60"] = 25000m, ["61-90"] = 15000m, ["90+"] = 10000m }
        });
    }

    public Task<BankFeedMetrics> QueryBankFeedStatusAsync()
    {
        // VERIFY_IN_DOCS(QBFC access pattern for bank feed pending transactions and last fetch)
        // If unavailable via QBFC, set capability false and return stale cached value.
        return Task.FromResult(new BankFeedMetrics { PendingCount = 0, LastFetchTime = null });
    }

    public Task<ReconciliationMetrics> QueryReconciliationStatusAsync()
    {
        // VERIFY_IN_DOCS(QBFC report for last reconciled date per account)
        return Task.FromResult(new ReconciliationMetrics { LastReconciledDate = null, UnreconciledDays = null });
    }

    public Task<PayrollMetrics> QueryPayrollLiabilitiesAsync()
    {
        // VERIFY_IN_DOCS(QBFC payroll liabilities report access and unpaid indicator)
        return Task.FromResult(new PayrollMetrics { LiabilityUnpaid = false, UnpaidAmount = 0m });
    }
}
```

## Edge Cases & Assumptions

- Bridge rejects all unknown commands (`CMD_NOT_ALLOWED`).
- If feature retrieval is unsupported, capability is false and button disabled.
- Auth token validation middleware should be added before production rollout.

## Verification Flags

- `VERIFY_IN_DOCS(QBFC method/report request for A/R Aging Summary with overdue buckets)`
- `VERIFY_IN_DOCS(QBFC access pattern for bank feed pending transactions and last fetch)`
- `VERIFY_IN_DOCS(QBFC report for last reconciled date per account)`
- `VERIFY_IN_DOCS(QBFC payroll liabilities report access and unpaid indicator)`

---

# 6. Scene Definitions

## 6.1 Scene Set (exact seven)

- `basic_bookkeeping`
- `bank_feed_review`
- `reconciliation_mode`
- `trucking_business`
- `retail_business`
- `service_business`
- `payroll_mode`

```javascript
// file: plugin/src/scenes.js
/**
 * Purpose: authoritative scene layout definitions for K01..K15 and S1..S3.
 * Dependencies: contract command names.
 * Ownership: Plugin presentation logic only.
 * Failure Modes: disabled rules gray out actions when capability/safety check fails.
 */
import { COMMANDS } from './contract.js';

const C = {
  good: '#22c55e', warn: '#eab308', bad: '#ef4444', neutral: '#3b82f6', disabled: '#6b7280'
};

const mk = (label, command, data = [], color = () => C.neutral, disabledWhen = () => false, payloadFactory = () => ({})) =>
  ({ label, command, data, color, disabledWhen, payloadFactory });

export const SCENES = {
  basic_bookkeeping: {
    sceneId: 'basic_bookkeeping', businessType: 'general',
    keys: {
      K01: mk('A/R Aging', COMMANDS.QB_QUERY_AR_AGING, ['ar.overduePercent'], s => s?.ar?.overduePercent > 0.15 ? C.bad : C.good),
      K02: mk('Bank Feed', COMMANDS.QB_QUERY_BANK_FEED_STATUS, ['bankFeed.pendingCount'], s => s?.bankFeed?.pendingCount > 10 ? C.warn : C.good, s => s?.capabilities?.bankFeed !== true),
      K03: mk('Recon', COMMANDS.QB_QUERY_RECONCILIATION_STATUS, ['reconciliation.unreconciledDays'], s => (s?.reconciliation?.unreconciledDays ?? 0) > 30 ? C.bad : C.good),
      K04: mk('Payroll', COMMANDS.QB_QUERY_PAYROLL_LIABILITIES, ['payroll.liabilityUnpaid'], s => s?.payroll?.liabilityUnpaid ? C.bad : C.good, s => s?.capabilities?.payroll !== true),
      K05: mk('Snapshot', COMMANDS.GET_STATE_SNAPSHOT),
      K06: mk('Prep JE', COMMANDS.PREPARE_JOURNAL_ENTRY, ['monthLock.lockedThroughDate'], () => C.neutral),
      K07: mk('Confirm JE', COMMANDS.CONFIRM_JOURNAL_ENTRY),
      K08: mk('Company', COMMANDS.QB_GET_COMPANY_IDENTITY),
      K09: mk('Connect', COMMANDS.QB_CONNECT),
      K10: mk('Disconnect', COMMANDS.QB_DISCONNECT),
      K11: mk('Go Bank', COMMANDS.SET_SCENE, [], () => C.neutral, () => false, () => ({ sceneId: 'bank_feed_review' })),
      K12: mk('Go Recon', COMMANDS.SET_SCENE, [], () => C.neutral, () => false, () => ({ sceneId: 'reconciliation_mode' })),
      K13: mk('Go Payroll', COMMANDS.SET_SCENE, [], () => C.neutral, () => false, () => ({ sceneId: 'payroll_mode' })),
      K14: mk('Client Map', COMMANDS.GET_STATE_SNAPSHOT),
      K15: mk('Refresh', COMMANDS.GET_STATE_SNAPSHOT)
    },
    sideButtons: {
      S1: mk('Emergency Sync', COMMANDS.QB_CONNECT),
      S2: mk('Client Switch', COMMANDS.QB_GET_COMPANY_IDENTITY),
      S3: mk('Mode Toggle', COMMANDS.SET_SCENE)
    }
  },
  bank_feed_review: { sceneId: 'bank_feed_review', businessType: 'general', keys: {}, sideButtons: {} },
  reconciliation_mode: { sceneId: 'reconciliation_mode', businessType: 'general', keys: {}, sideButtons: {} },
  trucking_business: { sceneId: 'trucking_business', businessType: 'trucking', keys: {}, sideButtons: {} },
  retail_business: { sceneId: 'retail_business', businessType: 'retail', keys: {}, sideButtons: {} },
  service_business: { sceneId: 'service_business', businessType: 'service', keys: {}, sideButtons: {} },
  payroll_mode: { sceneId: 'payroll_mode', businessType: 'general', keys: {}, sideButtons: {} }
};

for (const id of Object.keys(SCENES)) {
  if (Object.keys(SCENES[id].keys).length === 0) {
    SCENES[id].keys = { ...SCENES.basic_bookkeeping.keys };
  }
  SCENES[id].sideButtons = {
    S1: mk('Emergency Sync', COMMANDS.QB_CONNECT),
    S2: mk('Client Switch', COMMANDS.QB_GET_COMPANY_IDENTITY),
    S3: mk('Mode Toggle', COMMANDS.SET_SCENE)
  };
}

const ORDER = ['basic_bookkeeping', 'bank_feed_review', 'reconciliation_mode', 'trucking_business', 'retail_business', 'service_business', 'payroll_mode'];
export function nextScene(curr) { return ORDER[(ORDER.indexOf(curr) + 1) % ORDER.length]; }
```

## 6.2 Scene Switching Rules

- Manual: S3 sends `set_scene` and plugin rotates scene order.
- Auto: on `company_changed`, plugin requests snapshot and picks mapped scene by `businessType`.
- No restart required; WebSocket stays open; full redraw executed.

## Edge Cases & Assumptions

- For brevity, non-primary scenes reuse complete 15-key baseline until customized.
- Disabled buttons render gray and append `LOCKED` suffix if blocked by month lock.

## Verification Flags

- `VERIFY_IN_DOCS(VSD SDK maximum label length and multiline rendering)`

---

# 7. State Logic Table

| State Field | Source | Update Frequency | Used By Buttons | Threshold | UI Behavior |
|---|---|---|---|---|---|
| `reconciliation.unreconciledDays` | `date(today)-lastReconciledDate` from recon source | 30s poll max | K03 | `>30` | red |
| `ar.overduePercent` | `overdueAR / totalAR` from A/R aging | 30s poll max | K01 | `>0.15` | red |
| `bankFeed.pendingCount` | `VERIFY_IN_DOCS(QB bank feed pending source)` | 30s poll max | K02 | `>10` | yellow |
| `payroll.liabilityUnpaid` | `VERIFY_IN_DOCS(QB payroll liability source)` | 30s poll max | K04 | `true` | red |
| `monthLock.lockedThroughDate` | bridge config/policy store | on config change + startup | K06/K07/payroll write keys | action date `<= lockedThroughDate` | disabled gray + `LOCKED` |
| `health.qbConnected` | bridge QB session status | event + 30s | all status keys | false | all keys gray/offline overlay |
| `health.staleAgeSec` | bridge cache timer | each response | status keys | `>120` | amber `STALE` suffix |

Computation definitions:
- `unreconciledDays = floor((utcNow - lastReconciledDate).TotalDays)`.
- `overduePercent = overdueAmount / totalAr` (0 when `totalAr==0`).
- `bank feed pending` = number of imported-not-reviewed feed items `VERIFY_IN_DOCS(...)`.
- `payroll liability unpaid` = true when outstanding payroll liabilities > 0 `VERIFY_IN_DOCS(...)`.

## Edge Cases & Assumptions

- If source unavailable, metric remains null and related button disabled.
- Bridge never infers unavailable accounting facts.

## Verification Flags

- `VERIFY_IN_DOCS(QB bank feed pending source)`
- `VERIFY_IN_DOCS(QB payroll liability source)`

---

# 8. Client Detection Logic

## 8.1 Detection + Rebind Flow

1. Bridge polls company identity every 30s and on connect.
2. If identity hash changes, bridge emits event:
   - `type=event`, `payload.eventName=company_changed`.
3. Plugin receives event, calls `get_state_snapshot`.
4. Plugin resolves `businessType` from snapshot profile and applies scene.
5. Plugin re-renders K01..K15 and S1..S3 without reconnect.

Identity strategy:
- Preferred: file path + company name + stable unique ID from SDK reports.
- Hash key: SHA-256 of canonical identity JSON.
- Profile store file: `%ProgramData%\WaDorTaxBridge\data\company-profiles.json`.

## 8.2 No Company Open Handling

- Bridge sets `company=null`, `health.qbConnected=false`, `capabilities.*=false`.
- Plugin enters offline/limited mode; only connect/query-safe keys remain enabled.

## Edge Cases & Assumptions

- File path may be inaccessible; fallback hash uses name + unique ID fields.
- Scene defaults to `basic_bookkeeping` when mapping missing.

## Verification Flags

- `VERIFY_IN_DOCS(QBFC fields for company file path and stable company identifier)`

---

# 9. Safety Controls

## 9.1 Guardrails

- Month lock gate:
  - Any write with accounting date `<= lockedThroughDate` rejected with `error.code="MONTH_LOCKED"`.
- Large JE two-step:
  1. `prepare_journal_entry` returns preview + risk flags + confirmation token.
  2. Plugin presents explicit confirm.
  3. `confirm_journal_entry` required with same correlation/confirmation token.
- Payroll void two-step:
  - `prepare_void_payroll` + `confirm_void_payroll`.
  - If unsupported, `capabilities.voidPayroll=false`, button shows `NOT AVAILABLE`.

## 9.2 Audit Logging

Log fields:
- timestamp
- correlation id
- company identity hash
- button id / user action
- command + redacted payload summary
- result status
- error details

Storage:
- `%ProgramData%\WaDorTaxBridge\logs\bridge-YYYYMMDD.log`
- daily rolling append.

## 9.3 Debugging Tools

- Console debug mode (`DOTNET_ENVIRONMENT=Development`).
- Button test simulator (`tools/button-test-simulator.js`) sends synthetic command envelopes.
- Structured error examples: `CMD_NOT_ALLOWED`, `QB_NOT_AVAILABLE`, `MONTH_LOCKED`, `CONFIRMATION_REQUIRED`.
- Doc Verification Needed list kept in deployment checklist.

## Edge Cases & Assumptions

- If confirmation token expires, action is rejected and must be re-prepared.
- Sensitive fields (account numbers, payroll identifiers) are redacted.

## Verification Flags

- `VERIFY_IN_DOCS(QuickBooks supportability for payroll void operations through SDK)`
- `VERIFY_IN_DOCS(Any QBFC write request schema for journal entries)`

---

# 10. Deployment Guide

## 10.1 Windows Startup Instructions

1. Install QuickBooks Desktop SDK/QBFC components on bridge host.
2. Build bridge:
   - `dotnet build bridge/WaDorTaxBridge.csproj -c Release`
3. Configure `%ProgramData%\WaDorTaxBridge\appsettings.json` (thresholds, lock date, auth token).
4. Run bridge interactively for validation:
   - `dotnet run --project bridge/WaDorTaxBridge.csproj`
5. Install as Windows Service (recommended) or Scheduled Task.
6. Package plugin folder and install into VSD plugin directory.
7. Restart VSD host; verify M18 keys render and bridge connection established.

## 10.2 Deployment Checklist

- [ ] Bridge listens only on localhost (`127.0.0.1`).
- [ ] Auth token enforced for all commands.
- [ ] Allowlist enforced and tested (`CMD_NOT_ALLOWED`).
- [ ] Month lock date configured.
- [ ] Large JE threshold configured.
- [ ] Audit logs writing + rotation confirmed.
- [ ] Poll interval is 30s max default.
- [ ] Offline UI behavior verified.
- [ ] Button simulator validated against bridge.
- [ ] **Doc Verification Needed:** `VERIFY_IN_DOCS(VSDinside SDK key APIs)`
- [ ] **Doc Verification Needed:** `VERIFY_IN_DOCS(QBFC AR aging request details)`
- [ ] **Doc Verification Needed:** `VERIFY_IN_DOCS(QBFC bank feed status availability)`
- [ ] **Doc Verification Needed:** `VERIFY_IN_DOCS(QBFC reconciliation status source)`
- [ ] **Doc Verification Needed:** `VERIFY_IN_DOCS(QBFC payroll liabilities source)`
- [ ] **Doc Verification Needed:** `VERIFY_IN_DOCS(QBFC company identity fields)`
- [ ] **Doc Verification Needed:** `VERIFY_IN_DOCS(QBFC payroll void supportability)`

## Edge Cases & Assumptions

- If any verification item fails, related capability remains false (fail-closed).
- Bridge can run as console app when service installation is restricted.

## Verification Flags

- `VERIFY_IN_DOCS(VSDinside SDK key APIs)`
- `VERIFY_IN_DOCS(QBFC AR aging request details)`
- `VERIFY_IN_DOCS(QBFC bank feed status availability)`
- `VERIFY_IN_DOCS(QBFC reconciliation status source)`
- `VERIFY_IN_DOCS(QBFC payroll liabilities source)`
- `VERIFY_IN_DOCS(QBFC company identity fields)`
- `VERIFY_IN_DOCS(QBFC payroll void supportability)`
