<!-- SACRED DOCUMENT — DO NOT MODIFY except for checkmarks ([ ] → [x]) and review findings. -->
<!-- You MUST NEVER alter, revert, or delete files outside the scope of this plan. -->
<!-- Plans in docs/plans/ are PERMANENT artifacts. There are ZERO exceptions. -->

# Collection Info Exposure + Agent-Controlled Bulk Scrolling

## User Story 1: Expose CollectionInfo/CollectionItemInfo in the accessibility tree

Virtualized lists (`RecyclerView`, Compose `LazyColumn`/`LazyGrid`) only materialize accessibility
nodes for on-screen (plus a small buffer) items. `AccessibilityNodeInfo.getCollectionInfo()` on the
container and `getCollectionItemInfo()` on rendered items are the only signal of an item's true
position and the container's full size — this lets an MCP client infer it is looking at "rows 3-8 of
50" instead of scrolling blind, and lets it choose how many times to bulk-scroll (User Story 2).

**Acceptance Criteria**
- [x] `AccessibilityNodeData` carries nullable `rowCount`/`columnCount`/`rowIndex`/`columnIndex`
- [x] `get_screen_state` TSV `flags` column includes `rows=N`/`cols=N` tokens on collection
      containers and `row=N`/`col=N` tokens on collection items
- [x] Collection containers with no other filterable attribute are no longer dropped by
      `shouldKeepNode`
- [x] `docs/MCP_TOOLS.md` flags reference, legend, and node-filtering description reflect the new
      tokens/behavior

### Task 1.1: Extract CollectionInfo/CollectionItemInfo in AccessibilityTreeParser

**File**: `app/src/main/kotlin/com/danielealbano/androidremotecontrolmcp/services/accessibility/AccessibilityTreeParser.kt`

**Action** (modify): add nullable fields + KDoc to `AccessibilityNodeData`.

```kotlin
 * @property webRole Chromium DOM role from `getExtras()` (e.g., "link", "heading", "article"),
 *   populated by Chrome and the Android System WebView on every web accessibility node. Null for
 *   native and Compose nodes. Used to scope WebView node collapsing to web content only.
 * @property targetUrl Target URL from `getExtras()` for links and images, or null when absent.
 * @property rowCount Total row count from [AccessibilityNodeInfo.getCollectionInfo] when this
 *   node is a collection container (e.g., RecyclerView, LazyColumn). May exceed the number of
 *   currently rendered child items — virtualized lists only materialize on-screen items as
 *   accessibility nodes. Null when the node is not a collection container.
 * @property columnCount Total column count from CollectionInfo, alongside [rowCount]. Null when
 *   the node is not a collection container.
 * @property rowIndex Row index from [AccessibilityNodeInfo.getCollectionItemInfo] when this node
 *   is an item within a collection container. Null when the node is not a collection item.
 * @property columnIndex Column index from CollectionItemInfo, alongside [rowIndex]. Null when the
 *   node is not a collection item.
 * @property children The child nodes of this node.
 */
@Serializable
data class AccessibilityNodeData(
    val id: String,
    val className: String? = null,
    val text: String? = null,
    val contentDescription: String? = null,
    val resourceId: String? = null,
    val bounds: BoundsData,
    val clickable: Boolean = false,
    val longClickable: Boolean = false,
    val focusable: Boolean = false,
    val scrollable: Boolean = false,
    val editable: Boolean = false,
    val enabled: Boolean = false,
    val visible: Boolean = false,
    val webRole: String? = null,
    val targetUrl: String? = null,
    val rowCount: Int? = null,
    val columnCount: Int? = null,
    val rowIndex: Int? = null,
    val columnIndex: Int? = null,
    val children: List<AccessibilityNodeData> = emptyList(),
)
```

**Action** (modify): extract the new values in `parseNode`, right after `targetUrl`.

```kotlin
            val extras = node.extras
            val webRole = extras?.getString(EXTRA_KEY_CHROME_ROLE)?.takeIf(String::isNotEmpty)
            val targetUrl = extras?.getString(EXTRA_KEY_TARGET_URL)?.takeIf(String::isNotEmpty)

            // CollectionInfo/CollectionItemInfo are set by list/grid widgets (RecyclerView,
            // Compose LazyColumn/LazyGrid) on the container node and its rendered item nodes
            // respectively. Negative values are the platform's undefined-value convention.
            val collectionInfo = node.collectionInfo
            val rowCount = collectionInfo?.rowCount?.takeIf { it >= 0 }
            val columnCount = collectionInfo?.columnCount?.takeIf { it >= 0 }
            val collectionItemInfo = node.collectionItemInfo
            val rowIndex = collectionItemInfo?.rowIndex?.takeIf { it >= 0 }
            val columnIndex = collectionItemInfo?.columnIndex?.takeIf { it >= 0 }

            // Max depth protection: return current node as leaf without recursing into children
```

**Action** (modify): pass the new values into the max-depth leaf-node construction.

```kotlin
                val leafNode =
                    AccessibilityNodeData(
                        id = nodeId,
                        className = className,
                        text = text,
                        contentDescription = contentDescription,
                        resourceId = resourceId,
                        bounds = bounds,
                        clickable = clickable,
                        longClickable = longClickable,
                        focusable = focusable,
                        scrollable = scrollable,
                        editable = editable,
                        enabled = enabled,
                        visible = visible,
                        webRole = webRole,
                        targetUrl = targetUrl,
                        rowCount = rowCount,
                        columnCount = columnCount,
                        rowIndex = rowIndex,
                        columnIndex = columnIndex,
                        children = emptyList(),
                    )
```

**Action** (modify): pass the new values into the normal `nodeData` construction.

```kotlin
            val nodeData =
                AccessibilityNodeData(
                    id = nodeId,
                    className = className,
                    text = text,
                    contentDescription = contentDescription,
                    resourceId = resourceId,
                    bounds = bounds,
                    clickable = clickable,
                    longClickable = longClickable,
                    focusable = focusable,
                    scrollable = scrollable,
                    editable = editable,
                    enabled = enabled,
                    visible = visible,
                    webRole = webRole,
                    targetUrl = targetUrl,
                    rowCount = rowCount,
                    columnCount = columnCount,
                    rowIndex = rowIndex,
                    columnIndex = columnIndex,
                    children = children,
                )
```

**Definition of Done**
- [x] `AccessibilityNodeData` has the 4 new nullable fields with KDoc
- [x] Both `parseNode` construction sites (leaf and normal) populate them
- [x] Negative CollectionInfo/CollectionItemInfo values map to `null`

### Task 1.2: Surface new fields in CompactTreeFormatter TSV output

**File**: `app/src/main/kotlin/com/danielealbano/androidremotecontrolmcp/services/accessibility/CompactTreeFormatter.kt`

**Action** (modify): extend `shouldKeepNode` so collection containers survive filtering even with
no other filterable attribute.

```kotlin
        internal fun shouldKeepNode(node: AccessibilityNodeData): Boolean =
            !node.text.isNullOrEmpty() ||
                !node.contentDescription.isNullOrEmpty() ||
                !node.resourceId.isNullOrEmpty() ||
                node.clickable ||
                node.longClickable ||
                node.scrollable ||
                node.editable ||
                node.rowCount != null ||
                node.columnCount != null
```

**Action** (modify): update the class-level KDoc's "KEPT if ANY of" list (currently ends at "is
editable") to add the new criterion, keeping it consistent with the code above:

```kotlin
 * Nodes are filtered: a node is KEPT if ANY of:
 * - has non-null, non-empty text
 * - has non-null, non-empty contentDescription
 * - has non-null, non-empty resourceId
 * - is clickable
 * - is longClickable
 * - is scrollable
 * - is editable
 * - is a collection container (has a non-null rowCount or columnCount)
 *
 * Filtered nodes are skipped in output, but their children are still
 * walked and may appear if they independently pass the filter.
 */
```

**Action** (modify): append collection tokens in `buildFlags`, after the existing `ena` block.

```kotlin
                if (node.enabled) {
                    append(FLAG_SEPARATOR)
                    append(FLAG_ENABLED)
                }
                if (node.rowCount != null) {
                    append(FLAG_SEPARATOR)
                    append("rows=${node.rowCount}")
                }
                if (node.columnCount != null) {
                    append(FLAG_SEPARATOR)
                    append("cols=${node.columnCount}")
                }
                if (node.rowIndex != null) {
                    append(FLAG_SEPARATOR)
                    append("row=${node.rowIndex}")
                }
                if (node.columnIndex != null) {
                    append(FLAG_SEPARATOR)
                    append("col=${node.columnIndex}")
                }
            }
```

**Action** (modify): update `buildFlags`'s KDoc order comment (currently "Order: on/off, clk, lclk,
foc, scr, edt, ena") to append `, rows, cols, row, col`.

**Action** (modify): extend the flags legend note.

```kotlin
            const val NOTE_LINE_FLAGS_LEGEND =
                "note:flags: on=onscreen off=offscreen clk=clickable lclk=longClickable " +
                    "foc=focusable scr=scrollable edt=editable ena=enabled " +
                    "rows=N/cols=N=container's total row/column count (may exceed rendered " +
                    "items) row=N/col=N=item's position within its container"
```

**Action** (modify): update the class-level KDoc's "Line 3" description (currently `- Line 3:
note:flags: on=onscreen off=offscreen clk=clickable lclk=longClickable foc=focusable
scr=scrollable edt=editable ena=enabled`) to append the same `rows=N/cols=N.../row=N/col=N...`
text used in `NOTE_LINE_FLAGS_LEGEND` above.

**Definition of Done**
- [x] Collection containers with only `rowCount`/`columnCount` set are kept by `shouldKeepNode`
- [x] `buildFlags` appends `rows=`/`cols=`/`row=`/`col=` tokens only when the corresponding field
      is non-null
- [x] Legend note, class KDoc "KEPT if ANY of" list, class KDoc "Line 3" description, and
      `buildFlags` order comment all describe the new tokens/criterion

### Task 1.3: Update docs/MCP_TOOLS.md

**File**: `docs/MCP_TOOLS.md`

**Action** (modify): extend the flags legend note line (currently `- \`note:flags: on=onscreen
off=offscreen clk=clickable lclk=longClickable foc=focusable scr=scrollable edt=editable
ena=enabled\``) to match the new `NOTE_LINE_FLAGS_LEGEND` text from Task 1.2.

**Action** (modify): the `android_get_screen_state` example response JSON (~line 239) embeds the
full legend verbatim inside its `"text"` string (`...note:flags: on=onscreen off=offscreen
clk=clickable lclk=longClickable foc=focusable scr=scrollable edt=editable ena=enabled\n...`).
Update that embedded substring to match the new `NOTE_LINE_FLAGS_LEGEND` text too, so the example
still matches real tool output.

**Action** (modify): extend the "Flags Reference" table with four new rows after `ena`:

```markdown
| `rows=N` | container's total row count (CollectionInfo; may exceed rendered items) |
| `cols=N` | container's total column count (CollectionInfo) |
| `row=N`  | item's row index within its container (CollectionItemInfo) |
| `col=N`  | item's column index within its container (CollectionItemInfo) |
```

Add one sentence above/below the table noting these four are `key=value` tokens (unlike the
boolean flags above them) and are only present on collection containers/items.

**Action** (modify): update the "Node Filtering" section (currently states nodes are omitted when
not clickable/longClickable/scrollable/editable and have no text/desc/resourceId) to add the
collection-container exception: a node with a non-null `rowCount` or `columnCount` is kept even if
none of the other criteria apply.

**Definition of Done**
- [x] Legend note (both the bulleted description and the embedded `get_screen_state` example
      response JSON), Flags Reference table, and Node Filtering section in `docs/MCP_TOOLS.md`
      match the implemented behavior

---

## User Story 2: Let the agent bulk-scroll by a specified count

Today, reaching a target several screens away requires the MCP client to issue one `android_scroll`
call per screen and re-inspect `get_screen_state` after each one — many round trips. With User Story
1's `rows=`/`row=` metadata now visible in `get_screen_state`, the client can already estimate how
many screens away a target is; it just needs a way to act on that estimate in one call instead of
many. Adding an optional `count` parameter to the existing `android_scroll` tool does exactly that:
the client picks the count from row-index data it already has and observes the result, which is
simpler and more robust than a server-side distance heuristic that has to detect and correct its
own estimation errors.

`android_scroll_to_node` (`app/src/main/kotlin/com/danielealbano/androidremotecontrolmcp/mcp/tools/NodeActionTools.kt`)
is intentionally left unchanged by this plan — it remains the right tool for final homing on a known
off-screen node after a bulk `android_scroll`.

**Acceptance Criteria**
- [x] `android_scroll` accepts an optional `count` parameter (default 1, range 1-50) that repeats
      the scroll gesture that many times before returning
- [x] Each repetition uses the same validated `direction`/`amount`/`variance`; a settle delay
      separates repetitions
- [x] A failure on any repetition stops the loop and reports a `CallToolResult(isError = true)`
      exactly as today's single-scroll failure does (via `McpToolUtils.handleActionResult`)
- [x] `count = 1` (the default) preserves today's exact response text — no regression for existing
      callers
- [x] `docs/MCP_TOOLS.md` and `docs/PROJECT.md` document the new parameter

### Task 2.1: Add `count` parameter to ScrollTool

**File**: `app/src/main/kotlin/com/danielealbano/androidremotecontrolmcp/mcp/tools/TouchActionTools.kt`

**Action** (modify): add `import kotlinx.coroutines.delay` to the file's import list, immediately
before `import kotlinx.serialization.json.JsonObject` (correct alphabetical position: `coroutines`
sorts before `serialization`).

**Action** (modify): update the class KDoc above `ScrollTool`.

```kotlin
/**
 * MCP tool handler for `scroll`.
 *
 * Scrolls the screen in a specified direction, optionally repeated `count` times in a single
 * call so a client can cover many rows/screens (e.g. using the `rows=`/`row=` CollectionInfo
 * flags from `get_screen_state`) without one round trip per screen. At `count = 50` this call
 * can take 15+ seconds (49 settle delays plus gesture time) — an accepted trade-off of doing
 * the distance estimate client-side instead of many round trips.
 *
 * **Input**: `{ "direction": "up"|"down"|"left"|"right", "amount": "small"|"medium"|"large",
 * "count": 1-50 }`
 * **Output**: `{ "content": [{ "type": "text", "text": "Scroll down (medium) executed" }] }`
 */
```

**Action** (modify): read and validate `count` in `execute`, right after the existing `variance`
validation, and delegate execution to a new `repeatScroll` helper (keeps `execute` under
detekt's `LongMethod` threshold instead of growing it further).

```kotlin
            McpToolUtils.validateNonNegative(variance, "variance")
            if (variance > MAX_VARIANCE) {
                throw McpToolException.InvalidParams(
                    "Parameter 'variance' must be between 0 and ${MAX_VARIANCE.toInt()}, got: $variance",
                )
            }

            val count = McpToolUtils.optionalInt(arguments, "count", 1)
            McpToolUtils.validatePositiveRange(count.toLong(), "count", MAX_COUNT.toLong())

            val variancePercent = variance / PERCENT_DIVISOR

            Log.d(TAG, "Executing scroll ${direction.name} with amount ${amount.name}, variance $variance%, count $count")
            return repeatScroll(direction, amount, variancePercent, count, directionStr, amountStr)
        }

        /**
         * Performs [count] scroll gestures via [ActionExecutor.scroll], settling
         * [REPEAT_SETTLE_DELAY_MS] between (but not after) repetitions. Reuses
         * [McpToolUtils.handleActionResult] per repetition so its existing
         * PermissionDenied-vs-ActionFailed distinction applies to every repetition, not just the
         * first. Extracted from [execute] to keep it under detekt's LongMethod threshold.
         */
        private suspend fun repeatScroll(
            direction: ScrollDirection,
            amount: ScrollAmount,
            variancePercent: Float,
            count: Int,
            directionStr: String,
            amountStr: String,
        ): CallToolResult {
            val message =
                if (count == 1) {
                    "Scroll ${directionStr.lowercase()} (${amountStr.lowercase()}) executed"
                } else {
                    "Scroll ${directionStr.lowercase()} (${amountStr.lowercase()}) executed $count times"
                }
            var lastResult: CallToolResult = McpToolUtils.textResult(message)
            repeat(count) { iteration ->
                val result = actionExecutor.scroll(direction, amount, variancePercent)
                lastResult = McpToolUtils.handleActionResult(result, message)
                if (iteration < count - 1) {
                    delay(REPEAT_SETTLE_DELAY_MS)
                }
            }
            return lastResult
        }
```

**Action** (modify): add the `count` property to the tool's input schema in `register`, after the
`variance` property block.

```kotlin
                                putJsonObject("count") {
                                    put("type", "integer")
                                    put(
                                        "description",
                                        "Number of times to repeat the scroll (1-$MAX_COUNT). " +
                                            "Use this to cover many rows/screens in one call, " +
                                            "e.g. when get_screen_state's rows=/row= flags " +
                                            "indicate the target is far from the currently " +
                                            "visible items.",
                                    )
                                    put("default", 1)
                                    put("minimum", 1)
                                    put("maximum", MAX_COUNT)
                                }
```

**Action** (modify): update the tool `description` string in `register` to mention repetition.

```kotlin
                description =
                    "Scrolls in the specified direction, optionally repeated `count` times in " +
                        "a single call. Applies random variance to scroll distance and center " +
                        "point for more natural-looking gestures. Returns after all " +
                        "repetitions complete.",
```

**Action** (modify): add the new constants to the companion object.

```kotlin
        companion object {
            const val TOOL_NAME = "scroll"
            private const val TAG = "MCP:ScrollTool"
            private const val PERCENT_DIVISOR = 100f
            private val DEFAULT_VARIANCE = ActionExecutor.DEFAULT_SCROLL_VARIANCE_PERCENT * PERCENT_DIVISOR
            private val MAX_VARIANCE = ActionExecutor.MAX_SCROLL_VARIANCE_PERCENT * PERCENT_DIVISOR
            private const val MAX_COUNT = 50
            private const val REPEAT_SETTLE_DELAY_MS = 300L
        }
```

**Definition of Done**
- [x] `count` defaults to 1, is validated to the 1-50 range via `McpToolUtils.validatePositiveRange`
- [x] Loop stops immediately (propagating the thrown `McpToolException`) on the first failed
      repetition, matching today's single-scroll error behavior
- [x] `count = 1` produces byte-identical response text to today's implementation
- [x] A `REPEAT_SETTLE_DELAY_MS` delay separates repetitions but does not follow the last one
- [x] `execute` stays under detekt's `LongMethod` threshold via the `repeatScroll` extraction (no
      new `@Suppress`)

### Task 2.2: Update docs/MCP_TOOLS.md and docs/PROJECT.md

**File**: `docs/MCP_TOOLS.md`

**Action** (modify): add a `count` row to the `android_scroll` Input Schema table (after
`variance`):

```markdown
| `count` | integer | No | 1 | Number of times to repeat the scroll (1-50). Use with the `rows=`/`row=` CollectionInfo flags from `get_screen_state` to cover many rows/screens in one call. |
```

**Action** (modify): add one sentence to the `android_scroll` description paragraph noting the
repeat behavior, and add `Invalid count (< 1 or > 50)` to the **Invalid params** error case bullet.

**File**: `docs/PROJECT.md`

**Action** (modify): update the `android_scroll` row in the "Touch Action Tools" table (Optional
Params column) to add `` `count` (int: 1-50, default 1) `` alongside the existing `amount`/
`variance` entries.

**Definition of Done**
- [x] `docs/MCP_TOOLS.md` and `docs/PROJECT.md` `android_scroll` entries document the `count`
      parameter and its validation range

---

## Testing

### Task T1: AccessibilityTreeParser tests

**File**: `app/src/test/kotlin/com/danielealbano/androidremotecontrolmcp/services/accessibility/AccessibilityTreeParserTest.kt`

**Setup**: extend the existing `createMockNode` helper with optional `collectionInfo:
AccessibilityNodeInfo.CollectionInfo? = null` and `collectionItemInfo:
AccessibilityNodeInfo.CollectionItemInfo? = null` params, wired via `every { node.collectionInfo }
returns collectionInfo` / `every { node.collectionItemInfo } returns collectionItemInfo`.

| Test | Verifies |
|------|----------|
| `parseNode extracts rowCount and columnCount from CollectionInfo` | Container node's CollectionInfo populates `rowCount`/`columnCount` on `AccessibilityNodeData` |
| `parseNode extracts rowIndex and columnIndex from CollectionItemInfo` | Item node's CollectionItemInfo populates `rowIndex`/`columnIndex` |
| `parseNode leaves collection fields null when CollectionInfo and CollectionItemInfo are absent` | Regression guard: non-collection nodes get null values |
| `parseNode maps negative CollectionInfo and CollectionItemInfo values to null` | `rowCount`/`columnCount`/`rowIndex`/`columnIndex` of -1 map to null |
| `parseNode extracts collection fields on max-depth leaf nodes` | Leaf-node branch (depth >= `MAX_TREE_DEPTH`) also populates the new fields |

### Task T2: CompactTreeFormatter tests

**File**: `app/src/test/kotlin/com/danielealbano/androidremotecontrolmcp/services/accessibility/CompactTreeFormatterTest.kt`

| Test | Verifies |
|------|----------|
| `buildFlags appends rows and cols tokens for collection container` | Container with `rowCount=50, columnCount=1` produces flags ending `,rows=50,cols=1` |
| `buildFlags appends row and col tokens for collection item` | Item with `rowIndex=3, columnIndex=0` produces flags ending `,row=3,col=0` |
| `buildFlags omits collection tokens when fields are null` | Regression guard: no `rows=`/`cols=`/`row=`/`col=` tokens appear |
| `shouldKeepNode returns true for collection container with no other filterable attributes` | Node with only `rowCount`/`columnCount` set (no text/desc/resourceId/clickable/longClickable/scrollable/editable) is kept |
| `shouldKeepNode returns false for plain container without collection info` | Regression guard: unchanged behavior for non-collection structural nodes |
| `format includes rows=N/cols=N legend text in flags note line` | `NOTE_LINE_FLAGS_LEGEND` output contains the new legend text |

### Task T3: ScrollTool tests

**File**: `app/src/test/kotlin/com/danielealbano/androidremotecontrolmcp/mcp/tools/TouchActionToolsTest.kt` (`ScrollToolTests` nested class)

| Test | Verifies |
|------|----------|
| `scroll with default count executes exactly once` | Regression guard: `actionExecutor.scroll` invoked exactly 1 time; response text unchanged (`"Scroll down (medium) executed"`, no "times" suffix) |
| `scroll with count 3 executes three times and reports count in message` | `actionExecutor.scroll` invoked exactly 3 times; response text contains `"3 times"` |
| `scroll with count below 1 throws InvalidParams` | `count = 0` rejected by `validatePositiveRange` |
| `scroll with count above 50 throws InvalidParams` | `count = 51` rejected |
| `scroll stops after first failed repetition` | With `count = 5` and the 2nd call returning `Result.failure`, `actionExecutor.scroll` is invoked exactly 2 times and `McpToolException.ActionFailed` is thrown |
| `scroll surfaces PermissionDenied when accessibility service unavailable mid-repeat` | `handleActionResult`'s existing "not available" → `PermissionDenied` mapping still applies inside the repeat loop |

The settle delay's exact timing/count (`REPEAT_SETTLE_DELAY_MS` called `count - 1` times) is not
separately asserted — it is implied by the "executes exactly N times" tests above (`runTest`
auto-advances virtual time, so it does not block them) and is simple enough to verify by reading
the implementation during code review.

---

## Final Quality Gates (run once, after both user stories are implemented)

- [x] `make lint` — zero warnings/errors
- [x] `make test-unit` — 1878/1881 pass; see Review Notes below for the 3 pre-existing failures
- [x] `./gradlew :app:build` — builds without errors/warnings; see Review Notes below for scope
- [x] `code-reviewer` subagent run in plan compliance mode — zero unresolved CRITICAL/WARNING findings

## Review Notes

- **Task 2.1 detekt refactor**: `ScrollTool.repeatScroll` ended up with 4 params
  (`direction`, `amount`, `variancePercent`, `count`) instead of the plan's 6 — it derives
  display strings via `direction.name.lowercase()`/`amount.name.lowercase()` instead of taking
  separate `directionStr`/`amountStr` params, and the schema properties were extracted into
  `buildInputSchemaProperties()`. Both changes were required to stay under detekt's
  `LongParameterList`/`LongMethod` thresholds and are behaviorally identical (verified: every
  valid enum's `name.lowercase()` equals its canonical input token, and the `count=1` regression
  test asserts the exact original response text).
- **3 pre-existing test failures** (`make test-unit`): `NgrokTunnelIntegrationTest` (missing
  `NGROK_AUTHTOKEN`/`.env`), `CloudflareTunnelIntegrationTest` (needs network egress to
  Cloudflare), `NetworkUtilsTest` loopback test (host network-interface topology). None of these
  files are touched by this plan's diff; accepted as known environment gaps.
- **Build gate scoped to `:app:build`**: the root `./gradlew build` also runs `e2e-tests`, which
  requires a rootful Podman socket (documented as a project prerequisite, missing in this
  environment). `:app:build` — assemble + lint + detekt + ktlint across all variants — succeeded
  cleanly, which is what this plan's code changes affect.
