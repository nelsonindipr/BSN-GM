# Imported Players Implementation Plan

This plan describes how to evolve BSN GM’s current `isImport` flag into a fuller imported-player system modeled after the BSN and common FIBA-league roster rules.


## External Rule Assumptions

This design uses the current BSN-inspired baseline of three imports per team, while keeping the rules configurable because FIBA domestic leagues vary widely in how they classify foreign, heritage, naturalized, regional-quota, and game-day eligible players.

## Goals

- Keep the current BSN feel: three active imports, visible import roster slots, salary differences, and substitution pressure.
- Add a league-rules layer so future seasons can mimic BSN, FIBA Europe-style non-domestic limits, Asian-player exceptions, naturalized-player handling, and playoff lock rules without hard-coding every rule inside signing/trade screens.
- Make imported-player decisions strategic: scouting, registration, temporary injury replacements, playoff eligibility, chemistry/culture fit, and market timing.
- Preserve quick-play simplicity by shipping rules in phases and keeping defaults close to the current game.

## Current Code Baseline

The game already has the core concept of imported players:

- `MAX_IMPORTS = 3` and `ROSTER_MAX = 12` define roster limits.
- `loadBSNRosters()` assigns `isImport`, `country`, and import-aware salaries from official roster/import lists.
- The generated player factory supports `genPlayer(isImport, age, ovr)`, and free agents can be generated as imports.
- Signing blocks a user from exceeding the import limit and tracks regular-season/playoff import substitutions.
- CPU teams can fill empty import slots and replace weak imports from free agency.
- The roster UI shows import count, import dots, and import badges.

## Recommended Data Model

Add a small rules/data layer instead of only using `isImport`.

### Player Fields

Extend player objects with:

```js
{
  isImport: true,
  nationality: 'USA',
  passportCountry: 'USA',
  localStatus: 'local' | 'import' | 'heritage' | 'naturalized' | 'asian_quota' | 'cotonou',
  registrationStatus: 'unregistered' | 'active' | 'reserve' | 'released' | 'injury_replacement',
  firstRegisteredWeek: 0,
  lastRegisteredWeek: null,
  importSeasonTeamId: 'GUA',
  replacementForId: null,
  fibaClearance: true,
  workPermit: true,
  playoffEligible: false,
  acquiredAfterDeadline: false,
  marketRegion: 'NBA/G-League' | 'LATAM' | 'Europe' | 'Australia' | 'Caribbean',
  languageFit: 0,
  cultureFit: 0,
  travelFatigue: 0
}
```

Keep `isImport` as a derived/cache field during the transition so existing UI and sim code continue to work.

### League Rule Preset

Add a `LEAGUE_RULES` object that can support BSN now and other FIBA-style leagues later:

```js
const LEAGUE_RULES = {
  BSN: {
    activeImportLimit: 3,
    rosterImportLimit: 3,
    gameImportLimit: 3,
    maxRegularSeasonImportChanges: 6,
    maxPlayoffImportChanges: 2,
    hasTradeDeadline: true,
    hasPlayoffEligibilityDeadline: true,
    localStatusesCountingAsLocal: ['local', 'heritage'],
    allowNaturalizedAsLocal: false,
    salaryPremium: 1.8,
    replacementTypes: ['performance', 'injury', 'temporary_fiba_window']
  },
  FIBA_GENERIC: {
    activeImportLimit: 4,
    rosterImportLimit: 6,
    gameImportLimit: 4,
    maxRegularSeasonImportChanges: 3,
    maxPlayoffImportChanges: 0,
    hasTradeDeadline: true,
    hasPlayoffEligibilityDeadline: true,
    localStatusesCountingAsLocal: ['local', 'heritage', 'cotonou'],
    allowNaturalizedAsLocal: 'league_specific',
    salaryPremium: 1.6,
    replacementTypes: ['injury', 'performance']
  }
};
```

## Implementation Phases

### Phase 1 — Centralize Import Rule Checks

Create reusable helpers and route all roster moves through them.

Suggested helpers:

```js
function getLeagueRules() { return LEAGUE_RULES[G.leagueId || 'BSN']; }
function countsAsImport(player, rules = getLeagueRules()) { ... }
function teamImportCount(teamId, opts = { activeOnly: true }) { ... }
function canRegisterImport(teamId, player, context) { ... }
function canAcquirePlayer(teamId, player, context) { ... }
function registerPlayer(teamId, player, context) { ... }
function releaseOrReserveImport(teamId, playerId, reason) { ... }
```

Refactor these current paths to use the helpers:

- User signing from free agency.
- User trades.
- Draft selection.
- CPU free-agent management.
- CPU import replacement logic.
- Offseason re-signing/expired contracts.

Acceptance criteria:

- Existing `MAX_IMPORTS = 3` behavior remains unchanged for BSN.
- Every signing/trade/draft failure returns a consistent reason object for toast/news messages.
- No import-limit rule is duplicated inside individual UI handlers.

### Phase 2 — Registration and Active/Reserve Imports

Add an import registration layer so a team can own/recruit more players than it can activate, if a league preset allows it.

For BSN, keep it strict by default:

- Maximum three active/registered imports.
- Released imports go back to the free-agent pool.
- Optional “reserve import” mode can be disabled for BSN but available for other FIBA-style leagues.

For FIBA-style presets:

- A team may hold more foreign contracts than it can dress.
- Game-day roster validates `gameImportLimit`.
- Playoff rosters can lock at a specific week/date.

Acceptance criteria:

- Roster screen can distinguish `IMPORT ACTIVO`, `IMPORT RESERVA`, `LOCAL`, `HERITAGE`, and `NATURALIZADO`.
- Lineup/game simulation only uses eligible active players.
- Player modal shows nationality and registration status.

### Phase 3 — Import Market and Scouting

Create a dedicated import market experience separate from general free agency.

Features:

- Import market filters by region, position, OVR, salary ask, availability, and clearance risk.
- Player cards show “BSN fit” or “FIBA fit” based on style, role, travel, language, culture, and salary.
- Add signing risk: clearance delay, late arrival, conditioning, or buyout.
- Better imports can appear in waves around real calendar beats: preseason, NBA/G-League cuts, FIBA windows, deadline week, playoff reinforcements.

Suggested generation sources:

- NBA/G-League veterans: high OVR, expensive, high short-term impact.
- LATAM/Caribbean imports: strong cultural fit, medium salary, lower clearance risk.
- Europe/Australia imports: balanced ratings, may have buyout/timing issues.
- Young upside imports: cheaper, volatile development.

Acceptance criteria:

- Import free agents feel different from local free agents.
- Best imports are not always immediately available.
- CPU teams compete for the same limited import pool.

### Phase 4 — Substitutions, Deadlines, and Playoff Eligibility

Build an explicit transaction ledger for import moves.

Track:

- Import registrations used.
- Import substitutions used.
- Date/week of signing.
- Whether the move happened before or after the trade/import deadline.
- Whether the player is playoff eligible.
- Injury-replacement exception status.

Rules to support:

- BSN-style limit: three imports active, regular-season substitution cap, smaller playoff substitution cap.
- Deadline behavior: after deadline, block active import changes unless injury exception applies.
- Playoff lock: only registered/eligible imports can play in postseason unless the preset allows emergency replacement.

Acceptance criteria:

- Roster and free-agent screens show remaining import substitutions.
- Attempting to sign an ineligible playoff import gives a clear reason.
- CPU teams obey the same substitution/deadline rules as the user.

### Phase 5 — Naturalized, Heritage, and League-Specific Local Status

Implement a flexible local-status system.

Statuses:

- `local`: native/domestic player.
- `heritage`: eligible as local by league-specific ancestry/passport rule.
- `naturalized`: player acquired local nationality after a cutoff; may count as import depending on league.
- `asian_quota` / `cotonou`: league-specific exceptions for leagues that use regional categories.
- `import`: counts against import limits.

Game impact:

- Some players can be valuable because they have import-level ratings but count as local under certain rules.
- Scouting should highlight disputed/uncertain status.
- Admin ruling events can reclassify a player before the season.

Acceptance criteria:

- `countsAsImport()` is the single source of truth.
- Switching from `BSN` to `FIBA_GENERIC` rules can change classification without rewriting players.
- UI labels stay understandable in Spanish.

### Phase 6 — AI Strategy

Upgrade CPU behavior from “sign best import” to a strategic model.

CPU import logic should consider:

- Weakest active import.
- Position scarcity.
- Team phase: rebuilding, contending, injury crisis, playoff push.
- Import substitution budget remaining.
- Salary/cap pressure.
- Fit with team playing style.
- Deadline urgency.
- Whether a domestic replacement would preserve an import slot.

Acceptance criteria:

- Bad teams do not waste all substitutions early unless desperate.
- Contenders aggressively upgrade before the deadline.
- CPU teams reserve playoff substitutions when rules make that valuable.

### Phase 7 — UI/UX Changes

Recommended screens:

1. **Roster**
   - Import meter: active/import/reserve status.
   - Remaining substitutions.
   - Playoff eligibility badge.

2. **Mercado de Importados**
   - Dedicated tab or filter inside free agency.
   - Region, status, salary, fit, clearance risk.

3. **Player Modal**
   - Nationality, local status, import status, clearance, registered week, playoff eligibility.

4. **League Office / Rules**
   - Show rule preset and deadlines.
   - Explain what counts as an import.

5. **Transactions**
   - Add transaction types: `import_register`, `import_release`, `import_replace`, `clearance_delay`, `playoff_lock`.

Acceptance criteria:

- A user can always tell why a move is legal or blocked.
- Import strategy is visible without opening the code or reading external rules.

## Suggested File-Level Work Plan

Because this project currently keeps most code in `index.html`, implement in small slices:

1. Add `LEAGUE_RULES` and helper functions near the existing constants and payroll helpers.
2. Add migration defaults in `loadBSNRosters()` and `genPlayer()` so every player gets local-status fields.
3. Refactor `sign()`, `draftPick()`, and `doTrade()` to use `canAcquirePlayer()`.
4. Refactor `aiManageRoster()` to use the same rule helpers.
5. Update `renderRoster()`, `buildFARows()`, and `openPlayerModal()` with status badges.
6. Add transaction/news entries for import registration and blocked moves.
7. Add a smoke-test checklist because there is currently no automated test suite.

## Smoke-Test Checklist

After implementation, manually test these flows in the browser:

- Start a new BSN season and verify each team begins with at most three active imports.
- Try to sign a fourth import and confirm the move is blocked.
- Cut an import, sign another import, and verify substitution count changes only in the proper phases.
- Trade for an import while already at the limit and confirm the move is blocked.
- Advance past the deadline and confirm active import replacements are blocked or limited by exception.
- Start playoffs and confirm playoff substitution/eligibility rules apply.
- Confirm CPU teams do not exceed import limits after several simulated weeks.
- Confirm roster, free-agent, player modal, transaction log, and news all use the same import labels.

## Open Product Decisions

Before coding the full system, decide:

- Should BSN GM allow reserve imports, or should BSN stay three active imports only?
- Should a released import be able to sign with another BSN team immediately?
- Should injury replacements count against the substitution limit?
- Should “heritage” Puerto Rican diaspora players count as local by default?
- Should generated draft prospects ever be imports, or should imports only enter through the import market?
- Should import contracts be weekly/monthly, season-long, or standard yearly contracts?

## Recommended First PR

Start with Phase 1 only:

- Add rules helpers.
- Keep current behavior identical.
- Refactor user signing, draft, trade, and CPU import signings to use shared validation.
- Add clearer toasts/news for blocked import moves.

That first PR creates a safe foundation for the richer BSN/FIBA imported-player features without changing the core simulation balance all at once.
