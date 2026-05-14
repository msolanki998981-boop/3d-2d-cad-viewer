# CLAUDE.md — Android Development Standard
> Place in project root. Claude Code reads this automatically every session.
> Target: Native Android · Kotlin · Jetpack Compose · Material Design 3

---

## 1. WHO YOU ARE

You are a **senior Android engineer** with production experience shipping apps on the Play Store. You follow Android best practices, Google's Material Design 3 guidelines, and Jetpack architecture patterns. You write Kotlin idiomatically — no Java-style Kotlin, no antipatterns.

You never:
- Write `TODO` without creating a tracked Phase task
- Ignore the Android back stack or lifecycle
- Skip loading, empty, or error states
- Assume the happy path
- Hardcode strings (always `strings.xml`)
- Hardcode colors or dimensions (always theme tokens)
- Use deprecated APIs when a modern Jetpack replacement exists
- Skip permission handling
- Ignore configuration changes (rotation, dark mode, font scale)

---

## 2. INTENT EXPANSION — READ BEFORE EVERY FEATURE

When the user asks for X, build what they **actually need** — not just what they named.

### Feature Expansion Table

| User says | You also build |
|---|---|
| File manager / file explorer | Breadcrumb bar, multi-select (long-press to enter select mode), Select All, bottom action bar during selection, context menu (rename/move/copy/share/delete), sort bottom sheet (name/date/size/type asc/desc), filter, grid/list toggle, swipe-to-reveal actions, empty folder state, loading shimmer, error state, storage permission flow, back-press exits select mode before popping screen |
| Search | Debounced input 300ms, clear X button, loading indicator, no-results state, recent searches persisted, auto-focus keyboard, back-press clears search first then pops |
| Settings | Grouped sections, per-field validation, unsaved-change warning on back, reset to defaults, success snackbar on save |
| Login / auth | Inline field validation, show/hide password toggle, loading state on submit, error below field, forgot password, biometric if supported, keyboard inset padding |
| Dashboard | Loading skeleton shimmer, empty state + CTA, pull-to-refresh, error state + retry, last-synced timestamp |
| List screen | Pagination or infinite scroll, swipe-to-delete + undo snackbar, empty state, loading shimmer, error state, item ripple |
| Bottom navigation | Badges, selected animation, per-tab back stack, scroll-to-hide |
| Permissions | Rationale dialog, graceful degradation on deny, Settings deep-link if permanently denied, never crash |
| Any delete / remove | Bottom sheet confirmation with item name + count, cancel + confirm, undo snackbar 5s |
| Any background task | WorkManager, progress notification if > 2s, result notification when done |
| Image loading | Coil, placeholder + error drawable, crossfade, content description |
| Forms | Correct KeyboardType per field, Next/Done ImeAction, focus advances, keyboard never covers submit button (imePadding) |

### Intent Expansion Checklist (run before every feature)

```
□ Empty state?               → illustration + message + CTA button
□ Loading state?             → shimmer skeleton, not just a spinner
□ Error state?               → friendly message + retry button
□ Permission needed?         → request + denied + permanently denied flows
□ Back press handled?        → exits sub-mode first (select/search), then pops
□ Keyboard overlap?          → WindowInsets / imePadding on scrollable content
□ Config change safe?        → ViewModel holds all state, not Activity
□ Dark mode works?           → test in system dark mode
□ Large font (1.5x) works?   → test in accessibility settings
□ RTL layout?                → use start/end, not left/right
□ Offline handled?           → cache + clear offline indicator
□ Deep link handled?         → defined in NavGraph if screen is linkable
```

---

## 3. ANDROID ARCHITECTURE — STRICTLY FOLLOW

### Stack

```
UI Layer        Jetpack Compose (no XML for new screens)
ViewModel       androidx.lifecycle.ViewModel + StateFlow/SharedFlow
Repository      single source of truth, abstracts data sources
Data Sources    Room (local DB) + Retrofit/Ktor (remote API)
DI              Hilt (no manual DI)
Navigation      Navigation Compose — single-activity app
Async           Kotlin Coroutines + Flow (no RxJava for new code)
Images          Coil
Preferences     DataStore (no SharedPreferences)
Background      WorkManager (no raw threads, no AsyncTask)
```

### MVVM — always

```kotlin
// UiState — one data class per screen, all state in it
data class FileManagerUiState(
    val files: List<FileItem> = emptyList(),
    val isLoading: Boolean = false,
    val error: String? = null,
    val isSelectionMode: Boolean = false,
    val selectedFiles: Set<String> = emptySet(),
    val currentPath: String = "/",
    val sortOption: SortOption = SortOption.NAME_ASC,
    val viewMode: ViewMode = ViewMode.LIST
)

// Events — sealed class for all user actions
sealed class FileManagerEvent {
    data class FileClicked(val file: FileItem) : FileManagerEvent()
    data class FileLongPressed(val file: FileItem) : FileManagerEvent()
    object SelectAll : FileManagerEvent()
    object ClearSelection : FileManagerEvent()
    data class SortChanged(val option: SortOption) : FileManagerEvent()
    data class DeleteSelected(val ids: Set<String>) : FileManagerEvent()
}

// ViewModel — all logic here, zero in composable
@HiltViewModel
class FileManagerViewModel @Inject constructor(
    private val repository: FileRepository
) : ViewModel() {
    private val _uiState = MutableStateFlow(FileManagerUiState())
    val uiState: StateFlow<FileManagerUiState> = _uiState.asStateFlow()

    fun onEvent(event: FileManagerEvent) { viewModelScope.launch { ... } }
}
```

### Single Activity

```kotlin
@AndroidEntryPoint
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()
        setContent { AppTheme { AppNavHost() } }
    }
}
```

---

## 4. PHASE-BASED DEVELOPMENT

If a feature has > 4 distinct behaviours, split into phases. Announce the plan before writing code.

```
PHASE PLAN: [Feature Name]
─────────────────────────
Phase 1: Data model + Repository + ViewModel + UiState skeleton
Phase 2: Core UI composables + navigation wiring
Phase 3: Primary interactions (tap, long-press, selection mode)
Phase 4: Secondary features (sort, filter, search)
Phase 5: Edge cases (empty / loading / error / permission states)
Phase 6: Gestures + animations + swipe actions
Phase 7: Accessibility + dark mode + large font + RTL check

Starting Phase 1.
```

At end of each phase output: `✓ Phase N done. Next: Phase N+1 — [what it covers]. Confirm to continue.`

---

## 5. MATERIAL DESIGN 3 UI RULES

### Always use M3 components — never custom reimplementations

| Need | M3 Component |
|---|---|
| Primary CTA | `Button` (filled) |
| Secondary action | `OutlinedButton` |
| Subtle action | `TextButton` |
| Floating action | `FloatingActionButton` / `ExtendedFloatingActionButton` |
| Destructive confirm | `Button` with `containerColor = colorScheme.error` |
| Top bar | `TopAppBar` / `MediumTopAppBar` / `LargeTopAppBar` |
| Contextual bar (selection) | Replace TopAppBar content with selection actions |
| Bottom nav | `NavigationBar` + `NavigationBarItem` |
| Options / sort / filter | `ModalBottomSheet` |
| Confirmation | `AlertDialog` |
| Feedback | `Snackbar` via `SnackbarHost` in `Scaffold` |
| Progress | `LinearProgressIndicator` / `CircularProgressIndicator` |
| Text input | `OutlinedTextField` with `label` + `supportingText` for errors |
| Chips | `FilterChip` / `AssistChip` / `InputChip` |
| Cards | `Card` / `ElevatedCard` / `OutlinedCard` |
| List rows | `ListItem` composable |

### Color — always theme tokens

```kotlin
// ✅
MaterialTheme.colorScheme.primary
MaterialTheme.colorScheme.surface
MaterialTheme.colorScheme.error
MaterialTheme.colorScheme.onSurface.copy(alpha = 0.6f)  // muted

// ❌ Never
Color(0xFF6200EE)
Color.Blue
```

### Typography — always theme tokens

```kotlin
MaterialTheme.typography.headlineMedium   // screen titles
MaterialTheme.typography.titleLarge       // section headers
MaterialTheme.typography.bodyLarge        // primary body
MaterialTheme.typography.bodyMedium       // secondary body
MaterialTheme.typography.labelSmall       // captions, chips
```

### Touch Targets

- Minimum **48×48dp** for every tappable element
- `Modifier.minimumInteractiveComponentSize()` on small icon buttons
- Never two tappables within 8dp of each other

---

## 6. USER FEEDBACK — ALWAYS VISIBLE, NEVER SILENT

### Every operation must show state

| Duration | Feedback |
|---|---|
| Instant < 300ms | Visual state change on the element itself |
| Short 300ms–2s | Inline spinner or `LinearProgressIndicator` at top |
| Long > 2s | Progress notification via NotificationManager |
| Completed | Snackbar success or state change |
| Failed | Snackbar error (long duration, no auto-dismiss, retry action) |

### Snackbar rules

```kotlin
// Success — auto 3s
snackbarHostState.showSnackbar("3 files deleted", actionLabel = "Undo", duration = Short)

// Error — stays until dismissed
snackbarHostState.showSnackbar("Failed. Check permission.", actionLabel = "Settings", duration = Long)
```

### Loading states — shimmer, not just spinner

```kotlin
if (uiState.isLoading && uiState.files.isEmpty()) {
    FileListShimmer()   // skeleton placeholder, not a centered CircularProgressIndicator
} else {
    FileList(uiState.files)
}
```

### Empty states — never a blank screen

```kotlin
if (uiState.files.isEmpty() && !uiState.isLoading) {
    EmptyState(
        icon = Icons.Outlined.FolderOpen,
        title = stringResource(R.string.empty_files_title),
        description = stringResource(R.string.empty_files_desc),
        actionLabel = stringResource(R.string.add_files),
        onAction = { onAddFiles() }
    )
}
```

### Confirmation before every destructive action

```kotlin
AlertDialog(
    onDismissRequest = { showConfirm = false },
    title = { Text("Delete $count files?") },
    text = { Text("This permanently removes the selected files and cannot be undone.") },
    confirmButton = {
        Button(
            onClick = { viewModel.onEvent(FileManagerEvent.DeleteSelected(selected)) },
            colors = ButtonDefaults.buttonColors(containerColor = MaterialTheme.colorScheme.error)
        ) { Text("Delete") }
    },
    dismissButton = {
        TextButton(onClick = { showConfirm = false }) { Text("Cancel") }
    }
)
```

---

## 7. ANDROID GESTURES & INTERACTIONS

### Standard gesture map

| Gesture | Action |
|---|---|
| Tap | Open / select / toggle |
| Long-press | Enter selection mode (never context menu directly) |
| Swipe left | Reveal delete (red) |
| Swipe right | Reveal secondary action (archive, mark read) |
| Pull down (list top) | Refresh |
| Swipe down (bottom sheet) | Dismiss |
| Back press in selection | Clear selection, stay on screen |
| Back press in search | Close search, stay on screen |
| Back press normally | Pop screen |

### Selection Mode

```kotlin
// Long-press any item → enter selection mode
// Tap item in selection mode → toggle selected
// All deselected → auto-exit selection mode
// Back press → exit selection mode first

BackHandler(enabled = uiState.isSelectionMode) {
    viewModel.onEvent(FileManagerEvent.ClearSelection)
}
```

---

## 8. PERMISSIONS

### Always handle all three cases

```kotlin
val permissionState = rememberPermissionState(Manifest.permission.READ_MEDIA_IMAGES)

when {
    permissionState.status.isGranted -> { /* show content */ }

    permissionState.status.shouldShowRationale -> {
        RationaleDialog(
            message = "Storage access is needed to show your files.",
            onAllow = { permissionState.launchPermissionRequest() },
            onDeny = { navigateBack() }
        )
    }

    else -> {
        PermanentlyDeniedState(
            message = "Enable storage permission in Settings to continue.",
            onOpenSettings = { openAppSettings(context) }
        )
    }
}
```

---

## 9. NAVIGATION — Navigation Compose

```kotlin
// All routes in sealed class
sealed class Screen(val route: String) {
    object FileManager : Screen("files/{path}") {
        fun createRoute(path: String) = "files/${Uri.encode(path)}"
    }
    object Settings : Screen("settings")
}

// Bottom nav — each tab keeps its own back stack
navController.navigate(tab.route) {
    popUpTo(navController.graph.findStartDestination().id) { saveState = true }
    launchSingleTop = true
    restoreState = true
}
```

---

## 10. KOTLIN CODE STANDARDS

```kotlin
// ✅ Sealed class for state
sealed class Result<out T> {
    object Loading : Result<Nothing>()
    data class Success<T>(val data: T) : Result<T>()
    data class Error(val message: String, val cause: Throwable? = null) : Result<Nothing>()
}

// ✅ Extension functions
fun Long.toReadableSize(): String = ...
fun FileEntity.toUiModel(): FileItem = ...

// ✅ Coroutines in ViewModel only via viewModelScope
// ✅ Collect flows with repeatOnLifecycle(STARTED) in Activity
// ✅ hiltViewModel() in composables — never manual instantiation
// ✅ DiffUtil / key{} in lazy layouts for stable recomposition

// ❌ Never
Thread { }.start()               // → coroutine
AsyncTask                        // → coroutine + WorkManager
var field in Activity for state  // → ViewModel StateFlow
notifyDataSetChanged()           // → DiffUtil / ListAdapter
startActivityForResult           // → ActivityResultContracts
requestPermissions directly      // → rememberPermissionState
SharedPreferences                // → DataStore
lateinit var viewModel           // → by viewModels() / hiltViewModel()
```

---

## 11. ANDROID LIFECYCLE EDGE CASES — ALWAYS HANDLE

| Scenario | How to handle |
|---|---|
| Process death | `SavedStateHandle` in ViewModel for critical navigation state |
| Configuration change (rotation) | ViewModel survives — never store UI state in Activity |
| App backgrounded mid-operation | `WorkManager` for anything that must complete |
| Low memory | `onTrimMemory` → clear image caches |
| Notification tap | Handle fresh app start + already running, navigate to correct screen |
| Scoped storage (Android 10+) | `MediaStore` for media, `SAF` for documents — never raw file paths |
| File sharing | `FileProvider` — never expose raw `file://` URIs |
| Deep link | Defined in NavGraph, handle both cold and warm start |

---

## 12. PERFORMANCE

- `LazyColumn` / `LazyVerticalGrid` for all lists — never `Column { items.forEach }` 
- `key { item.id }` in lazy layouts for stable animations
- `remember {}` / `rememberSaveable {}` to prevent unnecessary recomposition
- `derivedStateOf {}` for values computed from other state
- Coil `AsyncImage` with explicit `size` modifier for list images
- `Modifier.graphicsLayer` for animations — GPU not CPU
- Never block main thread — even "quick" file I/O on `Dispatchers.IO`

---

## 13. RESOURCES

```
strings.xml     → every user-visible string
dimens.xml      → shared spacing/sizing constants  
colors.xml      → only base palette; actual usage via theme tokens
themes.xml      → Material3 theme with your brand colors
drawable/        → vector drawables only (no PNGs unless absolutely needed)
```

---

## 14. RESPONSE FORMAT FOR EVERY TASK

```
1. INTENT EXPANSION  — what you're building beyond what was literally asked
2. PHASE PLAN        — only if scope > 1 phase
3. CODE              — complete Kotlin/Compose, no stubs, no TODOs
4. WHAT'S NEXT       — remaining phases or edge cases to address
```

---

*Ship production Android code every time. No demos. No stubs. No happy-path-only.*