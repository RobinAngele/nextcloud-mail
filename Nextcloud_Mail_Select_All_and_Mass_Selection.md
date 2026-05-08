# Nextcloud Mail — Select All & Mass Selection Feature

## Student Documentation

---

## 1. Context

### What is Nextcloud Mail?

Nextcloud Mail is an open-source email client integrated into the Nextcloud platform.
It provides a web interface for reading, composing, and managing emails. The frontend
is a Vue 2 single-page application bundled with Webpack, backed by a PHP REST API.

### The problem

Before this contribution, users could only select messages **one at a time** or by
**shift-clicking within the currently loaded page** (~20 messages). There was no way to:

- Select all messages in a folder at once
- Select all messages matching a search or filter (e.g., all emails from a sender)
- Perform bulk actions (delete, move, archive) on more than one page of results

Three open issues had been requesting this for years:
- [#4285](https://github.com/nextcloud/mail/issues/4285) — "Select all messages in mailbox"
- [#7880](https://github.com/nextcloud/mail/issues/7880) — "Implement a Select All menu item"
- [#12149](https://github.com/nextcloud/mail/issues/12149) — "Important icon is missing from quick actions"

---

## 2. Architecture overview

### Key files

| File | Purpose |
|------|---------|
| `src/components/Mailbox.vue` | Parent component managing a single mailbox's envelope list |
| `src/components/EnvelopeList.vue` | Renders a list of envelope items with selection checkboxes |
| `src/components/MailboxThread.vue` | Top-level layout combining search bar + envelope list + thread view |
| `src/components/SearchMessages.vue` | Search bar with filter dialog (subject, date, sender, etc.) |
| `src/store/mainStore/actions.js` | Pinia store actions for API calls (fetch, delete, move) |
| `src/service/ThreadService.js` | Axios HTTP service for thread/message operations |

### Data flow before

```
Mailbox
  └── EnvelopeList (owns `selection[]` locally)
        └── Envelope × N (each has `flags.selected`)
```

Each `EnvelopeList` managed its own selection state. If messages were grouped by date
(Today, Yesterday, etc.), each group had an independent selection — shift-click couldn't
cross group boundaries.

### Data flow after

```
Mailbox (owns `selection[]` globally)
  ├── Select-all checkbox
  ├── EnvelopeList (receives `selection` as prop)
  │     └── Envelope × N
  └── EnvelopeList (receives `selection` as prop)
        └── Envelope × N
```

Selection state is centralized in `Mailbox`. Children receive it as a read-only prop
and emit changes upward via `update:selection` and `select-range` events.

---

## 3. Solution — Three stacked pull requests

The maintainer requested splitting the work into small, focused PRs so that bug fixes
could be backported to stable branches while features remain main-only.

### PR #12899 — Bug fixes (backportable)

**Branch:** `fix/important-icon-and-favorite`
**File:** `src/components/EnvelopeList.vue` (+8/-8)

#### Fix 1: Missing ImportantIcon (closes #12149)

The `ImportantIcon` component (filled label icon from `vue-material-design-icons/LabelVariant.vue`)
was used in the template of `EnvelopeList.vue` but never imported or registered:

```vue
<!-- Template (was already there) -->
<ImportantIcon :size="20" />
```

```javascript
// Script — was MISSING both the import and the component registration
import ImportantIcon from 'vue-material-design-icons/LabelVariant.vue'  // ← added

components: {
    ImportantIcon,  // ← added
    // ...
}
```

Without this, Vue 2 cannot resolve the component at runtime, resulting in a missing icon
during bulk selection.

#### Fix 2: Favorite/unfavorite logic

The methods were named counter-intuitively and used a flawed toggle logic:

```javascript
// BEFORE (broken)
favoriteAll() {
    const favFlag = !this.isAtLeastOneSelectedUnFavorite  // computes wrong value
    // ...
},
unFavoriteAll() {
    const favFlag = !this.isAtLeastOneSelectedFavorite     // computes wrong value
    // ...
}
```

**Bug:** When all selected messages are favorited, `isAtLeastOneSelectedUnFavorite`
is `false`, so `favFlag = true`. Clicking "Unfavorite" would try to favorite them again —
a no-op. The unfavorite action silently failed.

```javascript
// AFTER (fixed)
unfavoriteAll() {
    this.selectedEnvelopes.forEach((envelope) => {
        this.mainStore.markEnvelopeFavoriteOrUnfavorite({
            envelope,
            favFlag: false,  // always false — always unfavorites
        })
    })
    this.unselectAll()
},
favoriteAll() {
    this.selectedEnvelopes.forEach((envelope) => {
        this.mainStore.markEnvelopeFavoriteOrUnfavorite({
            envelope,
            favFlag: true,   // always true — always favorites
        })
    })
    this.unselectAll()
}
```

The template references were also renamed to match:
```vue
<!-- Unfavorite button → calls unfavoriteAll -->
<NcButton @click.prevent="unfavoriteAll"> ... </NcButton>

<!-- Favorite button → calls favoriteAll -->
<NcButton @click.prevent="favoriteAll"> ... </NcButton>
```

---

### PR #12900 — Select-all checkbox (main only)

**Branch:** `feat/select-all-checkbox`
**Files:** `Mailbox.vue` (+147/-1), `EnvelopeList.vue` (+74/-24)

#### Feature: Select-all checkbox

A checkbox is rendered at the top of the envelope list using `NcCheckboxRadioSwitch`
from `@nextcloud/vue`:

```vue
<NcCheckboxRadioSwitch
    :model-value="false"
    type="checkbox"
    class="select-all-bar"
    @update:checked="selectAll">
    {{ n('mail', 'Select {count} message', 'Select all {count} messages',
         flatEnvelopeList.length, { count: flatEnvelopeList.length }) }}
</NcCheckboxRadioSwitch>
```

CSS includes `margin-top: 8px` so the bar doesn't sit flush against the sticky search header.

#### Centralizing selection state

The `selection` array moves from `EnvelopeList` local data to `Mailbox` data:

```javascript
// Mailbox.vue — new data and methods
data() {
    return {
        selection: [],  // single source of truth
        // ...
    }
},
computed: {
    flatEnvelopeList() { /* all visible envelopes, regardless of grouping */ },
    selectMode() { return this.selection.length > 0 },
    allSelected() {
        return this.flatEnvelopeList.length > 0
            && this.selection.length === this.flatEnvelopeList.length
    },
},
methods: {
    selectAll() { this.selection = this.flatEnvelopeList.map(e => e.databaseId) },
    unselectAll() { this.selection = [] },
    onUpdateSelection(childSelection, childEnvelopes) {
        // Merge a child's selection into the global selection
        const childIds = new Set(childEnvelopes.map(e => e.databaseId))
        this.selection = this.selection.filter(id => !childIds.has(id))
        this.selection.push(...childSelection)
    },
    onSelectRange(from, to, deselect) {
        // Handle shift-click range selection across flat list
        const ids = this.flatEnvelopeList.slice(from, to + 1).map(e => e.databaseId)
        if (deselect) this.selection = this.selection.filter(id => !ids.includes(id))
        else this.selection = [...new Set([...this.selection, ...ids])]
    },
}
```

The `EnvelopeList` component is refactored to receive `selection` as a prop and emit
changes upward:

```javascript
// EnvelopeList.vue
props: {
    selection: { type: Array, default: () => [] },
    flatIndex: { type: Number, default: 0 },
},
watch: {
    selection: {
        handler(newSelection) {
            const set = new Set(newSelection)
            this.sortedEnvelops.forEach(env => {
                env.flags.selected = set.has(env.databaseId)
            })
        },
        immediate: true,
    },
},
methods: {
    emitLocalSelection() {
        const ids = this.sortedEnvelops.filter(e => e.flags.selected).map(e => e.databaseId)
        this.$emit('update:selection', ids, this.envelopes)
    },
    onEnvelopeSelectMultiple(envelope, index) {
        // Shift-click now emits global flat indices to parent
        this.$emit('select-range', this.flatIndex + lastToggledIndex, this.flatIndex + index)
    },
}
```

---

### PR #12901 — Filter-based mass selection (main only)

**Branch:** `feat/filter-mass-select`
**Files:** `SearchMessages.vue`, `MailboxThread.vue`, `Mailbox.vue`, `EnvelopeList.vue`

This is the final piece that completes #4285 and #7880.

#### Search modal button

A new button is added to the search dialog:

```javascript
// SearchMessages.vue
dialogButtons: [
    { label: t('mail', 'Clear'), callback: () => this.resetFilter(), type: 'secondary' },
    {
        label: t('mail', 'Select all matching'),  // ← NEW
        callback: () => this.selectAllMatching(),
        type: 'primary',
    },
    { label: t('mail', 'Search'), callback: () => this.closeSearchModal(), type: 'tertiary' },
],

methods: {
    selectAllMatching() {
        this.moreSearchActions = false  // close dialog
        this.$nextTick(() => {
            this.sendQueryEvent()                // emit search-changed
            this.$emit('select-all-matching')    // trigger mass selection
        })
    },
}
```

#### Event flow

```
SearchMessages
  │  @select-all-matching
  ▼
MailboxThread
  │  bus.emit('select-all-matching')
  ▼
Mailbox
  │  onBusSelectAllMatching()
  │    → loadEnvelopes()    // force fresh load with query
  │    → selectAllMatchingAction()
  │        → while (!endReached) await loadMore()   // fetch all pages
  │        → selection = flatEnvelopeList.map(...)   // select everything
```

#### Mass loading implementation

```javascript
// Mailbox.vue
async onBusSelectAllMatching() {
    this.loadingAllMatching = true   // show spinner
    this.selectAllMatching = true    // track state
    this.endReached = false          // reset pagination
    this.syncedMailboxes.delete(...) // clear cache to force reload
    await this.loadEnvelopes()       // fresh first page with query
    await this.selectAllMatchingAction()
},

async selectAllMatchingAction() {
    try {
        while (!this.endReached) {
            await this.loadMore()    // fetch next page
        }
        this.selection = this.flatEnvelopeList.map(e => e.databaseId)
    } finally {
        this.loadingAllMatching = false  // hide spinner
    }
},
```

#### Context-aware labels

The checkbox label adapts to the current context:

```javascript
selectAllLabel() {
    const count = this.flatEnvelopeList.length
    const hasFilter = this.searchQuery && this.searchQuery.trim() !== 'match:allof'

    if (hasFilter) {
        return this.n('mail', 'Select {count} matching message',
                            'Select {count} matching messages', count, { count })
    }
    if (!this.endReached && count > 0) {
        return this.n('mail', 'Select {count} loaded message',
                            'Select {count} loaded messages', count, { count })
    }
    return this.n('mail', 'Select {count} message',
                        'Select all {count} messages', count, { count })
},

selectAllHint() {
    if (!this.endReached && this.flatEnvelopeList.length > 0) {
        return this.t('mail',
            'Scroll down to include more messages, use filter to refine, ' +
            'or click an avatar circle to select one at a time')
    }
    return ''
},
```

#### Spinner and loading state

A separate row below the checkbox shows the loading indicator:

```vue
<NcCheckboxRadioSwitch
    :model-value="allSelected"
    :disabled="loadingAllMatching"     <!-- prevents double-trigger -->
    type="checkbox"
    @update:checked="allSelected ? unselectAll() : selectAll()">
    {{ selectAllLabel }}
</NcCheckboxRadioSwitch>

<!-- Loading row — separate element for proper alignment -->
<div v-if="loadingAllMatching" class="select-all-loading">
    <NcLoadingIcon :size="16" />
    <span>{{ t('mail', 'Selecting messages…') }}</span>
</div>

<!-- Hint row — shown when more messages exist -->
<div v-if="selectAllHint && !loadingAllMatching && !allSelected" class="select-all-hint">
    {{ selectAllHint }}
</div>
```

---

## 4. UX Decisions

### Why a checkbox instead of a menu item?

Original request #7880 suggested a "Select all" entry in the 3-dot menu. A
persistent checkbox is more discoverable — the user sees it immediately without
opening a menu. It also matches the mental model of "select all" in other email
clients (Gmail, Outlook).

### Why "loaded" instead of "first"?

The label says "Select 20 loaded messages" rather than "Select first 20 messages"
because messages are loaded incrementally as the user scrolls. After scrolling
down, 60 messages are loaded — "first 20" would be incorrect. "Loaded" is always
accurate.

### Why a hint row?

New users might not discover that they can select more than one page. The hint
*"Scroll down to include more messages, use filter to refine, or click an avatar
circle to select one at a time"* teaches them all three selection methods.

### Why the banner before loading all pages?

Loading all pages can be slow for large mailboxes. The blue banner *"All 20 visible.
Select all 80 matching?"* is an explicit confirmation step — the user must
opt in before triggering multiple API calls.

---

## 5. Security considerations

| Concern | Assessment |
|---------|-----------|
| **XSS (Cross-Site Scripting)** | No `v-html` or `innerHTML` used. All user-facing text uses `t()`/`n()` from `@nextcloud/l10n` which HTML-escapes output. |
| **CSRF** | All API calls go through `axios` configured by `@nextcloud/axios`, which attaches CSRF tokens. |
| **Injection** | The `searchQuery` string is only used for boolean checks (`!== 'match:allof'`) and never injected into the DOM. API calls use the existing store/service layer which properly encodes parameters. |
| **Authorization** | No new API endpoints. All operations reuse existing CRUD endpoints which already enforce mailbox ACLs. |
| **Denial of Service** | The mass-loading loop has a natural limit (stops when `endReached === true`). No new API endpoints are created that could be abused. |
| **Data exposure** | Selection state is local to the Vue component. No data is persisted or sent to external services. |
| **Event bus** | The `select-all-matching` bus event is internal to the Vue app (mitt), not exposed externally. |

---

## 6. Commit conventions

Per the repository's `AGENTS.md`, every commit must include:

```
AI-assisted: Cline (Claude)
Signed-off-by: RobinAngele <robin@robin4consulting.com>
```

Commits follow [Conventional Commits](https://www.conventionalcommits.org/):
- `fix:` for bug fixes (PR #12899)
- `feat:` for new features (PR #12900, #12901)

---

## 7. Testing

### Manual test plan

1. **Bug fix — ImportantIcon:** Select multiple messages. The "Mark as important"
   icon should appear in the multiselect header.

2. **Bug fix — Favorite/unfavorite:** Select all messages. Click Favorite → all
   become favorited. Click Unfavorite → all become unfavorited.

3. **Select-all checkbox:** Open a mailbox. A checkbox should appear at the top
   with the current page count. Click it → all visible messages selected.

4. **Shift-click range:** Select one message, hold Shift, click another in a
   different date group → all messages between are selected.

5. **Filter mass-select:** Open the search modal, set a filter. Click "Select all
   matching". The modal closes, a spinner appears, and all matching messages
   are loaded and selected.

6. **Context-aware labels:** Observe the checkbox label changes:
   - "Select N loaded messages" (default, more pages)
   - "Select N matching messages" (filter active)
   - "Select all N messages" (all pages loaded)

### Automated tests

No automated tests were added. The changes are purely frontend and rely on the
existing Vue component architecture. Integration tests for the selection flow
could be added in a follow-up PR using Playwright (the project's E2E framework).

---

## 8. Future improvements

### Backend bulk endpoint

The current mass-loading implementation iterates over all pages sequentially
(one API call per page). A backend endpoint accepting a filter query and
performing the bulk operation server-side would be significantly faster for
large mailboxes.

Suggested endpoint:
```
POST /apps/mail/api/mailbox/{id}/bulk
Body: { query: "subject:invoice start:1717977600", action: "delete" }
```

### Single unified multiselect header (#11526)
Currently each date group renders its own multiselect header. With selection
state centralized in `Mailbox`, a single header could be rendered at the
top of the list.

---

## 9. PR summary

| PR | Title | Scope | Files | Lines |
|----|-------|-------|-------|-------|
| [#12899](https://github.com/nextcloud/mail/pull/12899) | fix: ImportantIcon + favorite/unfavorite | Bug fixes | 1 | +8/-8 |
| [#12900](https://github.com/nextcloud/mail/pull/12900) | feat: select-all checkbox | Feature | 2 | +197/-24 |
| [#12901](https://github.com/nextcloud/mail/pull/12901) | feat: filter mass-select | Feature | 4 | +375/-26 |

Issues closed: #4285, #7880, #12149
Issues referenced: #6070, #7276, #11526, #9248
