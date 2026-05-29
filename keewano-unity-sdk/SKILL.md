---
name: keewano-unity-sdk
description: Use when working with the Keewano Unity SDK — integrating it, auditing instrumentation, adding event reporting, configuring consent, tracking purchases/ads/subscriptions/items/custom events/windows. Activates on references to `KeewanoSDK`, the `Keewano` namespace, `KeewanoSettings`, or files under `Assets/KeewanoSDK/` and `Assets/KeewanoCustomEvents/`. Teaches Claude to act as a proactive integration partner: survey the codebase, find gaps, propose atomic events whose stream reads like a player-behavior story.
---

# Keewano Unity SDK — Integration Guide

Lightweight Unity analytics SDK. Auto-tracks most player behavior, sends compact binary batches in the background. Your job: leave an event stream that **tells the story** of what the player did, so an AI agent or analyst reading it later can reconstruct the journey from events alone.

For method signatures: read `Assets/KeewanoSDK/Runtime/KeewanoSDK.cs`. This skill covers what the source can't — which moments to report, what to name them, how the stream should read.

---

## Mental model

Keewano is **not** Firebase / Amplitude / GameAnalytics. Those require each event to carry its own context as metadata. Keewano works the opposite way: **events are small and parameter-light; the backend reconstructs context from the temporal sequence.** You don't tag the "OK" click with `window="shop"` — `ReportWindowOpen("shop")` fired earlier; Keewano joins them on the timeline. **The history is the context.**

Sessions, funnels, balances, retention, churn — all reconstructed from the raw stream. Compact binary + background batching = fire events liberally; framerate is unaffected. Hundreds per session is normal.

- **Each action is its own event.** A Match3 level: enter → reset moves → 3× move-spent → win-reward. Five events, not one summary.
- **Don't pre-aggregate.** No `ReportLevelStats`. No `ReportSessionSummary`. Backend computes.
- **Don't bundle unrelated transactions.** One `ReportItemsExchange` = one transaction.
- **Custom events are moments, not summaries.** `ReportEnemyKilled("Dragon")` ✓. `ReportCombatSummary(kills, damage, time)` ✗.
- **State is derived, not sent.** Send deltas, not snapshots.

In doubt? Discrete action = discrete event.

Why the model looks this way: events are parameter-light and temporally sequenced because the consumer is an AI agent reading the stream, not a human writing SQL. Per-event timestamps + server-side ordering handle out-of-order delivery. Keewano's proprietary database rebuilds pivot tables, cohort analyses, funnel sequences, retention curves, and arbitrary metrics from the stream on demand at query time — fast enough that precomputation isn't required. Async events (subscription renewals, server-side validations) flow into the same per-user timeline. Raw event export is available. Parameter-light is a feature for AI consumption, not a constraint to work around.

---

## The stream is a story

Read later by an AI agent or analyst. Each event a sentence; the sequence the plot. Instrument so a cold reader can follow.

Good:

```
button_click       "open_shop"
window_open        "shop"
button_click       "buy_crystal_pack"
items_exchange     loc="shop"            from=[Coins, 500]    to=[Crystals, 100]
button_click       "close_shop"
window_close       "shop"
button_click       "play_level"
custom_event       LevelStart(5)
items_exchange     loc="level_entry"     from=[Life, 1]
items_reset        loc="level_start"     [Moves, 5]
items_exchange     loc="level_move"      from=[Moves, 1]                (×3)
items_exchange     loc="level_win"       to=[Coins, 200, XP, 50]
custom_event       LevelWon(5)
```

Reader reconstructs: shop visit, crystal pack purchase, level 5, won 200 coins + 50 XP. The level number lives in `LevelStart`/`LevelWon` parameters — the location strings (`level_entry`, `level_move`, `level_win`) stay stable across all levels.

Bad:

```
button_click       "btn_3"
window_open        "win_a7"
button_click       "btn_8"
custom_event       "result"   {id:5, moves:3, score:1200, gold:200}
```

Same data, no story.

### Naming

- **Action names, not widget names.** `"open_shop"` ✓, `"button_3"` ✗.
- **Consistent vocabulary.** `"shop"` everywhere; not `"shop"` here and `"store"` there.
- **Same item = same name.** `"Coins"` always; never `"Coins"`/`"coins"`/`"Gold Coins"`.
- **Specific over generic.** `PlayerDied("fall")` vs `PlayerDied("starvation")`. Not `PlayerDied()`.
- **One casing convention**, applied consistently.
- **No concatenated strings at runtime.** `$"level_{n}_move"`, `string.Format("enemy_{0}_killed", t)` create one unique string per value — bloats the backend, fragments aggregation, makes merge/split queries fail. Keep the string **stable**; put the variable in a custom event parameter (`LevelStart(5)`, `EnemyKilled("Dragon")`) or rely on surrounding scope context.

### GameObject names *are* event names

The SDK auto-reads `gameObject.name` for:
- Every Unity UI `Button` click.
- Every `KeewanoWindow` open/close.

Hierarchy name = analytics name. `Button (1)`, `Panel`, `Popup`, `GameObject` → useless. `BuyCrystalsBtn`, `ShopWindow` → self-documenting.

Defaults to rename: `Button`, `Button (1)`, `New Button`, `GameObject`, `Panel`, `Popup`, `Canvas`, `Image`, `Text`, `Test`, `Temp`, `NewWindow`.

Renaming would break references? Drop `KeewanoWindow` and call `ReportWindowOpen`/`ReportWindowClose` explicitly.

### Scopes: openers + closers bracket what happens inside

Most gameplay happens *inside* something — window, scene, level, encounter, ad, purchase. These are **scopes**. Every scope needs both endpoints. (Tutorials work differently — see below.)

Why scopes matter:
- **Attribution.** An `items_exchange` between `WindowOpen("shop")` and `WindowClose("shop")` is a shop purchase. Without brackets it's context-free.
- **Outcome.** A level scope closed with `LevelWon` vs `LevelLost` vs nothing tells three different stories. **Outcome = which closer fires**, not a parameter on one closer.
- **Durations & funnels.** Opener+closer enables time-in-scope, completion rate, drop-off step.

| Scope | Opener | Closer |
|---|---|---|
| Session, app, scene | auto | auto — scene names matter (no `SampleScene`) |
| Window / popup / panel | `KeewanoWindow` or `ReportWindowOpen` | same, paired by name |
| Level / mission | custom `LevelStart(id)` | custom `LevelWon`/`LevelLost`/`LevelQuit` |
| Encounter / quest | custom opener | custom outcome event |
| Ad opportunity | `ReportAdOffered` | `ReportAdRevenue` (+ optional items granted) or implicit decline |
| IAP | `ReportInAppPurchase` | `ReportInAppPurchaseItemsGranted` (may span time) |
| Subscription | `ReportSubscriptionRevenue` | next renewal/expiry |

Scopes nest. Session > scene > window > click. Reader infers nesting from temporal containment.

**Match opener and closer by ID.** Scope pairs that carry an identifier must use the same value on both ends: `LevelStart(5)` closes with `LevelWon(5)`, never `LevelWon(7)`. Same rule for windows (`WindowOpen("shop")` ↔ `WindowClose("shop")`), encounters, quests, IAP product IDs, ad placements, subscription package names. Mismatched IDs read as an orphan opener plus an orphan closer.

**Scope repetitions are auto-counted.** Backend knows how many times `LevelStart(5)` has fired for a given player — that's the attempt count. Don't add `attempt_number`, `session_count`, `times_opened`, `visit_count` parameters; the system reconstructs them from frequency.

**One-sided scopes** (opener without closer) are the most common audit finding — almost always an instrumentation bug. Propose the missing closer.

**Tutorials are not scopes.** They're milestone sequences. Each `ReportOnboardingMilestone` (or feature-tutorial `Tutorial<Feature>Milestone`) is a single event reported once per install. The funnel is built across the player base from which milestones each user reaches. Completion is just another milestone name (`"Tutorial Completed"`), not a "close" event paired with an opener. A user who drops out simply stops reporting further milestones — there's no closer to be missing.

---

## Your role

You're sitting with a developer integrating Keewano. Default to **proactively finding instrumentation gaps and proposing concrete edits** — not waiting for API questions. Methods are in `KeewanoSDK.cs`; discover them yourself.

On first contact, or any open-ended ask:

1. **Survey** using the [audit playbook](#integration-audit-playbook). Find subsystems present in code but absent from the stream — especially **windows**.
2. **Open with what's missing**, prioritized. Don't dump everything. Order: windows → IAP → FTUE → economy → custom events.
3. **Cite `file.cs:line`.** Show the diff. Offer to apply.
4. **Read the stream back in your head.** Could a stranger follow it? If not: more events, better names.
5. **Flag anti-patterns** in existing code.
6. **Ask for context** the code can't tell you ("where does validation land?", "tutorial as a controller or inline?").
7. **Lean toward more events.** They're cheap.
8. **Check the SDK version.** Read `SDK_VERSION` from `Assets/KeewanoSDK/Runtime/KNetwork.cs` (format: `Unity/X.Y.Z`). Compare against <https://github.com/Keewano/Keewano-UnitySDK/releases/latest>. If newer, offer to upgrade.

Stance: small, additive edits. Suggest one, get feedback, iterate.

---

## Setup

1. Import `KeewanoSDK.unitypackage` from [GitHub releases](https://github.com/Keewano/Keewano-UnitySDK/releases).
2. `Edit > Project Settings > Keewano` — paste API key from `app.keewano.com`.
3. *(GDPR-strict)* check **Require Player Consent**.

**No `Init()` call.** SDK auto-initializes via `[RuntimeInitializeOnLoadMethod]`. Don't create a `KeewanoSDK` GameObject; don't `AddComponent<KeewanoSDK>()`.

---

## Hard rules

1. **Never report IAP / ad / subscription revenue from a raw store callback.** Validate server-side first.
2. **`SetUserId` is permanent per install.** One call.
3. **Don't disable automatic button tracking** without a real reason.
4. **Test traffic: two complementary mechanisms.** Two projects (dev + prod) isolate environments at build time. `MarkAsTestUser` excludes specific users (internal team, QA on live builds, support) within the prod environment. See [Isolating test traffic](#isolating-test-traffic).
5. **Strings: non-null, non-empty, ≤256 chars** (except `LogError`, deep links, exceptions — unbounded).
6. **Same name = same thing, everywhere.**

---

## Automatic — don't manually re-report

App launch, scene load/unload, app pause/resume, Unity UI button clicks, unhandled exceptions, low memory, internet connectivity, platform/device/OS/RAM/VRAM/resolution/language, deep links, genuinity check, session boundaries.

---

## Patterns by domain

Methods are in `KeewanoSDK.cs`. This section is non-discoverable rules.

### IAP
- Only after server validation. Never from raw store callback.
- Pair `ReportInAppPurchase` + `ReportInAppPurchaseItemsGranted`. Same `productId`.
- **`ReportInAppPurchaseItemsGranted` is not substitutable** by `ReportItemsExchange` with a generic location like `"iap_purchase_grant"`. The specific API carries the `productId` that links items to revenue; without it, the backend can't compute revenue-per-item or product economics. Skipping it to "avoid double-counting" is wrong — the rule is use the right API for the source, not avoid both.
- Currency modes: USD cents (`uint`) or local (`float, isoCode`). Pick one project-wide.
- Pair can split across time for delayed/recurring grants (monthly pack: purchase once, grant each day).

### Subscriptions
- Same pattern as IAP. Report **every billing event** (initial, trial→paid, every renewal). Replay missed events at app launch.
- Same `packageName` on both calls.

### Ads
- Three-event scope: `ReportAdOffered` → `ReportAdRevenue` → `ReportAdItemsGranted`. Same `placement` across all three.
- Missing close = declined / failed. Valid story — don't fake.
- **`ReportAdItemsGranted` is not substitutable** by `ReportItemsExchange` with a generic location like `"rewarded_ad_gems"`. The specific API carries the `placement` that links items to ad revenue per placement; without it, ad ROI analysis breaks. Same rule for `ReportSubscriptionItemsGranted`.
- **Use the float overload for `ReportAdRevenue` even in USD.** Ad impression revenue is often sub-cent (`$0.005`, `$0.012`); the `uint cents` overload truncates to 0 or 1. Pass the actual decimal with `"USD"`: `ReportAdRevenue(placement, 0.005f, "USD")`.

### Items & virtual economy

**An "item" is anything countable that can be given to or taken from the player** — not just inventory objects. If it has a count and changes, it's an item.

- **Currencies**: Coins, Gems, Crystals, Tokens.
- **Discrete objects**: Sword, Potion, Character, Skin.
- **Stats**: XP, Score, Reputation.
- **Depletable / refillable resources**: Life, Energy, Stamina, Mana, Health, Hunger, Thirst, Oxygen.
- **Action counters**: Moves, Attempts.

Report every meaningful change — in either direction (gain, loss, decay, regen).

- `Keewano.Item(name, count=1)`. Same name everywhere — gameplay, IAP grants, ad rewards, support resets.
- `ReportItemsExchange(loc, from, to)` = one transaction. Don't bundle unrelated rewards.
- `ReportItemsReset(loc, items)` **sets** the balance — not a delta. For initialization / support corrections.
- One-sided exchange: `null` for the missing side (pure gain or pure loss).
- **`ReportItemsExchange` is for virtual economy ONLY.** Items granted from IAP / ads / subscriptions go through `ReportInAppPurchaseItemsGranted` / `ReportAdItemsGranted` / `ReportSubscriptionItemsGranted` — never `ReportItemsExchange`. Those APIs carry the productId / placement / packageName that links items to their revenue source; the backend needs that link to compute revenue-per-item, product economics, ad-placement ROI. A clever `loc` string like `"iap_purchase_grant"` is **not** a substitute — it loses the product link entirely.
- **`loc` describes WHY, not direction.** Direction is already in `from`/`to`. `"gems_earned"` / `"gems_spent"` is redundant — the from/to arrays say the same thing. Use the *event* causing the exchange: `"level_win_reward"`, `"shop_purchase"`, `"booster_use"`, `"daily_bonus"`, `"ad_reward"`, `"quest_reward"`.
- **The reason must come from the caller.** `AddGems(int)`, `SpendGems(int)`, `AddLife(int)`, `LoseLife(int)` don't know *why* on their own. Two valid fixes:
  1. **Pass `reason` from the caller** (preferred when there's an existing centralized analytics layer): `AddGems(int count, string reason)` → callers do `AddGems(reward, "level_win_reward")`, `AddGems(50, "daily_bonus")`. The helper still does all reporting; it just gets context from each caller. One-time refactor across call sites; doesn't break centralized analytics.
  2. **Move the `Report*` call to the caller** (when there's no centralized analytics layer to preserve). Helper just mutates state; the caller reports.

  Either way, the location string names the *event* (`"level_win_reward"`, `"shop_purchase"`), never the direction.

Examples beyond obvious currencies:
- Drink potion → `ReportItemsExchange("drink_potion", from=[Potion, 1], to=[Health, 50])`.
- Eat meal → `ReportItemsExchange("eat_meal", from=[Food, 1], to=[Hunger, 20])`.
- Hunger / energy decay tick → `ReportItemsExchange("hunger_tick", from=[Hunger, 1], to=null)` at whatever cadence the game measures it.
- XP from quest → `ReportItemsExchange("quest_reward", from=null, to=[XP, 100])`.
- Daily energy regen → `ReportItemsExchange("energy_regen", from=null, to=[Energy, 5])`.

### Custom events
- Define via `Keewano > Custom Events Editor`. **Never hand-edit `KeewanoSDK.CustomEvents.Generated.cs`.**
- One descriptive parameter. Three params = probably three events.
- Numeric IDs in the generated file are **internal**. Never surface in UI/logs/docs.
- **For new events:** write the JSON file(s) directly to `Assets/KeewanoCustomEvents/<EventName>.json`, then ask the developer to open `Keewano > Custom Events Editor` and click **Save**. Save triggers codegen — `Report<YourEvent>` appears in `Generated.cs`. Wait for confirmation, then add the `Report*` call sites. Adding calls before Save breaks the build.

### Tutorials
- `ReportOnboardingMilestone(name)` for first-launch FTUE.
- Custom `Tutorial<Feature>Milestone(step)` for later tutorials.
- Each milestone once per install.

### Identity & consent
- `SetUserId(uid)` at login (one-shot).
- `SetUserConsent(bool)` — only relevant if **Require Player Consent** is checked. Persists.
- `ReportUserRegisteredBeforeSDKIntegration(date)` once for veterans of pre-SDK player base. Past dates only.

### Errors
- Exceptions and `Debug.LogError` auto-captured. `LogError` for issues that surface as neither. Uncapped length.

### Attribution
- `ReportInstallCampaign(name)` once per install when source is known.

### A/B testing
- `ReportABTestGroupAssignment(testName, char)` at the moment the group is determined.

---

## Windows: every logical window must appear in the stream

Windows (shop, settings, inventory, pause, level select, popups, dialogs) are the **narrative anchors**. Without them, auto-button-clicks float without context. Highest-leverage gap in most audits.

**Rule:** every logical window produces paired `ReportWindowOpen("name") → ReportWindowClose("name")`.

The SDK ships `Assets/KeewanoSDK/Runtime/KeewanoWindow.cs`:

```csharp
[DisallowMultipleComponent]
public class KeewanoWindow : MonoBehaviour
{
    void OnEnable()  => KeewanoSDK.ReportWindowOpen(gameObject.name);
    void OnDisable() => KeewanoSDK.ReportWindowClose(gameObject.name);
}
```

Attach to every logical window root. Reports `gameObject.name` verbatim — **hierarchy name = analytics name** (see [GameObject names *are* event names](#gameobject-names-are-event-names)).

Use explicit `ReportWindowOpen`/`ReportWindowClose` calls when:
- Show/hide doesn't map to `OnEnable`/`OnDisable` (tween-out before disable).
- Window is a scene, not a panel.
- Window is an abstract state ("in checkout flow").

Non-Unity-UI buttons: `ReportButtonClick("action_name")` — action verb, not widget name.

---

## Isolating test traffic

Two complementary mechanisms — use both.

### 1. Two Keewano projects: dev environment vs prod environment

Two projects on `app.keewano.com` (prod + dev). Dev builds point at the dev project; prod builds at the prod project. Structural — QA / staging / internal builds never touch production data. Swap the API key in `KeewanoSettings.asset` per build config:

```csharp
#if UNITY_EDITOR
using UnityEditor;
using UnityEditor.Build;
using UnityEditor.Build.Reporting;

class KeewanoBuildPreprocessor : IPreprocessBuildWithReport
{
    public int callbackOrder => 0;

    public void OnPreprocessBuild(BuildReport report)
    {
        var settings = AssetDatabase.LoadAssetAtPath<KeewanoSettings>(
            "Assets/Resources/KeewanoSettings.asset");

        bool isDev = (report.summary.options & BuildOptions.Development) != 0;
        settings.APIKey = isDev ? "<dev-project-api-key>" : "<prod-project-api-key>";

        EditorUtility.SetDirty(settings);
        AssetDatabase.SaveAssets();
    }
}
#endif
```

Alternatives: multiple settings assets copied at build time; scripting defines; CI-injected keys.

### 2. `MarkAsTestUser`: exclude specific users within the prod environment

For users who hit the **production project** but should not count in production statistics — internal team running the live game, QA reproducing customer issues on a live build, support reps, paid testers on a release candidate.

```csharp
KeewanoSDK.MarkAsTestUser("qa_alice_" + SystemInfo.deviceName);
```

Not a substitute for the two-project split. The split keeps environments separate by design; `MarkAsTestUser` handles the in-production exception.

---

## Integration audit playbook

Grep these patterns, open the matched files, propose edits.

| Subsystem | Patterns | Suggest |
|---|---|---|
| **Logical windows** | UI panels, `SetActive(true)`/`(false)` on UI roots, dialog prefabs, modals | `KeewanoWindow` (preferred) or explicit `ReportWindowOpen`/`ReportWindowClose` |
| **Generic GameObject names** | UI Button GOs / `KeewanoWindow` roots named `Button`, `Button (1)`, `Panel`, `Popup`, `GameObject`, `Image`, `Test`, `Temp` | Rename to action/surface. Renaming disruptive → explicit calls with desired name |
| **Non-Unity-UI buttons** | Custom UI frameworks, 3D clickables, raycasts | `ReportButtonClick("action_name")` |
| **IAP** | `*IAP*`, `*Purchase*`, `*Store*`, `using UnityEngine.Purchasing`, `IStoreListener`, `ProcessPurchase` | After server validation: `ReportInAppPurchase` + `ReportInAppPurchaseItemsGranted` |
| **Subscriptions** | Same as IAP plus `subscription`, `vip`, `premium`, renewal logic | `ReportSubscriptionRevenue` + `ReportSubscriptionItemsGranted` per billing event |
| **Ads** | `*Ad*`, `*Rewarded*`, `IronSource`, `AppLovin`, `MAX`, `AdMob`, `ShowRewardedAd` | `ReportAdOffered` → `ReportAdRevenue` → `ReportAdItemsGranted` |
| **Tutorial / FTUE** | `*Tutorial*`, `*Onboarding*`, `*FTUE*`, `FirstTime*`, opening cutscenes | `ReportOnboardingMilestone` per step; custom `Tutorial<Feature>Milestone` for later tutorials |
| **Level / progression** | `*Level*`, `*Stage*`, `*Mission*`, `LoadScene("Level...")`, win/lose logic | Level scope: `LevelStart(id)` + `LevelWon`/`LevelLost`/`LevelQuit`. Inside: `ReportItemsExchange` / `ReportItemsReset` |
| **Items & virtual economy** | `*Shop*`, `*Inventory*`, `*Currency*`, `AddCoins`/`SpendGems`, XP / health / energy / mana / stamina / hunger / lives counters, regen / decay timers, level-up code | `ReportItemsExchange` / `ReportItemsReset` for every counted change, in either direction (gain, loss, decay, regen) |
| **Errors / save / network** | `try/catch` + `Debug.LogError`, save/load, network retry | `LogError` only for issues not auto-captured |
| **Identity** | Login, account, social auth | `SetUserId` once known |
| **Attribution** | Deep links, install referrer, attribution SDKs | `ReportInstallCampaign` |
| **Veterans** | Save-game `originalSignupDate`, cloud creation timestamps | `ReportUserRegisteredBeforeSDKIntegration` |
| **A/B testing** | Remote-config branching, experiment SDKs | `ReportABTestGroupAssignment` at group determination |

Unity IAP specifically: canonical hook is the server-validation callback. Flag reporting in `ProcessPurchase` without server round-trip.

### Custom events worth defining

- **Scope pairs** (always both endpoints): `LevelStart` + `LevelWon`/`LevelLost`/`LevelQuit`; `EncounterStart` + outcome; `QuestStarted` + completion/abandonment.
- **Feature tutorial milestones** (sequence, not a pair): one custom event `Tutorial<Feature>Milestone` with a String param. Report at each step — `"Tutorial Started"`, `"Learned Combo"`, `"Tutorial Completed"`. Each step once per install.
- **Single events**: `EnemyKilled(type)`, `BossDefeated(name)`, `BoosterUsed(type)`, `PowerupActivated(name)`, `CharacterSelected(name)`, `DifficultyChosen(level)`, `FriendInvited`, `GuildJoined`, `MessageSent`.

Narrow events with one parameter, always.

### Order for a fresh integration

1. Settings + dev/prod project split.
2. `SetUserId` at login. Veterans → `ReportUserRegisteredBeforeSDKIntegration`.
3. Verify dev-project routing: run an Editor session, confirm events land in the dev project (not prod).
4. Windows: `KeewanoWindow` on every logical window root; rename generic GameObjects.
5. IAP — highest revenue signal.
6. FTUE milestones.
7. Virtual economy.
8. Ads + subscriptions if applicable.
9. Domain custom events.
10. Attribution + consent.
11. Run a session, read the stream. Does it tell a story?

---

## Anti-patterns to flag

- **Logical windows without `WindowOpen`/`WindowClose`.** Biggest narrative gap.
- **Widget-name strings**: `ReportButtonClick("btn_3")`, `ReportWindowOpen("panel_a7")`.
- **Concatenated strings in Report calls**: `$"level_{n}_move"`, `string.Format("enemy_{0}_killed", type)`, `"buy_" + productId`. Bloats the backend with one unique string per value; aggregation fails. Move the variable into a custom event parameter; keep the string stable.
- **Mismatched IDs in scope pair**: `LevelStart(5)` closed by `LevelWon(7)`; `WindowOpen("shop")` closed by `WindowClose("store")`. Opener and closer must share the identifier.
- **Direction in `ReportItemsExchange` location strings** (`"gems_earned"`, `"gems_spent"`, `"lives_earned"`, `"life_spent"`). Direction is already encoded in `from`/`to`. Location must describe *why* the exchange happened (`"level_win_reward"`, `"shop_purchase"`, `"booster_use"`).
- **Generic state-mutation helpers reporting with a hardcoded location** (e.g., `AddGems(int)` calling `ReportItemsExchange("gems_earned", …)` with no caller-supplied reason). The helper can't know *why* without input from the caller — context is lost, location names degrade to direction-only. Either add a `reason` parameter to the helper signature (`AddGems(count, reason)`) or move the report to the caller. Don't leave a hardcoded generic `loc`.
- **Substituting `ReportItemsExchange` (with a "causal" `loc`) for `ReportInAppPurchaseItemsGranted` / `ReportAdItemsGranted` / `ReportSubscriptionItemsGranted`.** `loc="iap_purchase_grant"` and `loc="rewarded_ad_gems"` are *not* equivalents — they lose the `productId` / `placement` / `packageName` that links items to their revenue source. Revenue-per-item, product economics, and ad-placement ROI break. Use the specific Granted API for the source; don't skip it citing "double-count" concerns.
- **Manually tracked repetition counters**: `LevelStart(level=5, attempt=3)`, `WindowOpen("shop", visit=12)`. Backend computes scope frequency. Drop the counter.
- **Default Unity GameObject names** on auto-tracked elements (`Button`, `Button (1)`, `Panel`, `Popup`, `GameObject`, `Image`). Rename, or replace auto-tracking with explicit calls.
- **Inconsistent vocabulary** — `"shop"` here, `"store"` there; `"gold"` and `"Gold"`.
- **One-sided scopes** — opener without closer.
- **Generic scene names** (`SampleScene`, `Scene1`, `Scene (1)`). Rename the scene file.
- **IAP / ad / subscription revenue from raw store callback** without server validation.
- **Pre-aggregated summary events** (`ReportLevelStats`, `ReportSessionSummary`, `ReportDailyTotals`).
- **`new KeewanoSDK()`, `AddComponent<KeewanoSDK>()`, custom Init.**
- **Hand-edited `KeewanoSDK.CustomEvents.Generated.cs`.**
- **Adding `KeewanoSDK.Report<NewEvent>(...)` calls before the developer has clicked Save in the Custom Events Editor.** Writing the JSON alone doesn't generate the C# method — codegen only runs on Save. Until then the `Report*` method doesn't exist; the build fails. Write the JSON, ask the developer to open `Keewano > Custom Events Editor` and click Save, wait for confirmation, then add the calls.
- **Naming customer-side integration files with `Keewano*` prefix or placing them under `Assets/Keewano*/`.** That namespace belongs to the SDK. If a compile error appears in `Assets/KeewanoIntegration/Editor/KeewanoBuildPreprocessor.cs`, the developer can't tell at a glance whether the bug is in the SDK or in your own integration code — and will likely blame the SDK. Put customer-side helpers (build preprocessor, `KeewanoWindow` overrides, integration glue) under the customer's existing project folder (e.g., `Assets/<GameName>/Editor/`) and give them neutral names (`AnalyticsBuildPreprocessor.cs`, not `KeewanoBuildPreprocessor.cs`).
- **`SetUserId` with different IDs over time.**
- **`ReportItemsReset` used as a delta.** It sets; use `ReportItemsExchange` for deltas.
- **`disableButtonTracking` enabled** without a clear reason.
- **Manual reports of auto-captured events.**
- **Custom event IDs in customer-facing UI/logs/docs.** Internal only.
- **Wide-parameter custom events.** 3+ params = probably multiple events.
- **Stacking `MarkAsTestUser` on Editor / Development builds when the dev/prod project split is already in place.** With the build preprocessor swapping API keys, Editor and Dev builds already route to the dev project. Adding `MarkAsTestUser` on top tags those sessions as test users *inside the dev project too*, hiding them from the dev dashboard. `MarkAsTestUser` is for users hitting the **production project** (testers on release builds, internal team on live, support reps) — not a belt-and-suspenders layer on Editor sessions.
- **Mixing currency overloads inconsistently.** Each of IAP / subscription has `(name, uint cents)` and `(name, float, isoCode)`; pick one per revenue stream. **For ads, always use the float overload** — `uint cents` truncates sub-cent impression revenue (`$0.005`) to 0.
- **Bundled unrelated transactions** in one `ReportItemsExchange`.

---

## Pointers

- API key: `Assets/Resources/KeewanoSettings.asset`.
- Method surface: `Assets/KeewanoSDK/Runtime/KeewanoSDK.cs`.
- Docs: <https://keewano.github.io/Keewano-UnitySDK/>
- Source & releases: <https://github.com/Keewano/Keewano-UnitySDK>
- Dashboard: <https://app.keewano.com>
