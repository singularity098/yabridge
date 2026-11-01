# JUCE popup menus (e.g. Superior Drummer 3 dropdowns) close immediately with Wine 10+: root cause and fix

**TL;DR:** JUCE creates its popup menus as `WS_POPUP | WS_SYSMENU` windows. Wine counts a popup with a system menu as a *managed* window (`is_window_managed()` in `winex11.drv/window.c`), so instead of mapping the menu as an override-redirect popup it hands it to the window manager as a regular top-level window. With Wine 10.x and later the menu then closes again about 13 ms after it opens. Stripping `WS_SYSMENU` from captionless, non-resizable top-level popup windows before Wine creates their X11 window fixes it. A ~110-line patch to `src/wine-host/editor.{h,cpp}` is included below.

This likely explains the open reports in #467 (Superior Drummer 3 broken dropdown menus), #518 (no dropdown menus in EZDrummer 3 / Mix Monolith) and #399 (Hofa plugins, dropdowns disappear instantly), plus the related vstbridge report rations/vstbridge#11.

## Environment where this was diagnosed and verified

| | |
|---|---|
| yabridge | current `master` (`b580a9f7`), plus the patch below |
| Wine | 11.19 |
| Desktop | KDE Plasma 6.7.5, KWin on Wayland (REAPER runs under XWayland) |
| Host | REAPER (native Linux), plugin in the docked FX window |
| Plugins | Toontrack Superior Drummer 3 (VST3), EZdrummer 3 (VST3) |

## Symptom

Clicking a dropdown in Superior Drummer 3 (for example the drum kit selector) shows the menu for a split second and then it disappears. The known workarounds were KWin's *focus stealing prevention: extreme* window rule, a Wine virtual desktop, or Wine 10.0 or older.

## How the cause was found

I added temporary `WH_CBT`, `WH_CALLWNDPROC` and `WH_GETMESSAGE` hooks to the yabridge GUI thread to log what happens to the plugin's windows, then had the user do a single click on the kit dropdown. The trace (times in ms):

```
4372359  WM_LBUTTONDOWN on the plugin's JUCE window
4372361  create top-level window ... ('JUCE_…' "menu", style 0x86080000, exstyle 0x88)
4372367  four drop-shadow windows (exstyle 0x800a0) created and shown
4372380  WM_SHOWWINDOW(TRUE) to the "menu" window
4372393  all four shadow windows destroyed
4372396  WM_SHOWWINDOW(FALSE) to "menu", then menu destroyed
4372479  WM_LBUTTONUP
```

Between the menu appearing and being dismissed, there is no `WM_ACTIVATE`, `WM_KILLFOCUS`, `WM_CANCELMODE` or capture change. The yabridge editor stays the Win32 foreground window, and the mouse button is still held down. So the plugin (JUCE's `PopupMenu`) dismisses the menu itself. The roughly 13 ms delay matches the `lastFocusedTime + 10` check in JUCE's `MouseSourceState::checkButtonState()`.

The relevant detail is the menu's style: `0x86080000` = `WS_POPUP | WS_CLIPSIBLINGS | WS_CLIPCHILDREN | WS_SYSMENU`, with exstyle `WS_EX_TOPMOST | WS_EX_TOOLWINDOW`. In `is_window_managed()`:

```c
if (style & WS_POPUP)
{
    /* popup with sysmenu == caption are managed */
    if (style & WS_SYSMENU) return TRUE;
```

So the menu, its drop shadows and JUCE tooltips all become WM-managed `_NET_WM_WINDOW_TYPE_NORMAL` windows. That matches what xwayland-satellite#293 observed for these plugins' menus. With Wine 10+'s reworked window-state and `_NET_ACTIVE_WINDOW` tracking, KWin handling the menu as a real window is enough for JUCE to decide the menu should go away. Every known workaround (KWin "extreme" focus stealing prevention, a virtual desktop, older Wine) stops the window manager from acting on these windows.

## The fix

While an editor is open, yabridge installs a `WH_CBT` hook on its GUI thread. On `HCBT_CREATEWND` it removes `WS_SYSMENU` from new top-level windows that are:

- not `WS_CHILD`, and are `WS_POPUP`,
- without a full caption (`(style & WS_CAPTION) != WS_CAPTION`) and without `WS_THICKFRAME`,
- without `WS_EX_APPWINDOW`.

`NtUserCreateWindowEx()` re-reads the window style right after calling the CBT hook (`win = get_win_ptr(hwnd)` followed by `init_window_info`). It also does so before the initial `WindowPosChanging`, where winex11 creates the X11 window and decides `managed`. So the change takes effect before the X11 window exists, and Wine maps these windows as override-redirect popups like any native application's menus. Regular plugin windows and dialogs, which have a caption or a resizable frame, are not touched. The hook is reference-counted across editors, can be disabled with `YABRIDGE_NO_POPUP_FIX=1`, and logs each changed window when `YABRIDGE_DEBUG_LEVEL` contains `+editor`.

Verified result: Superior Drummer 3's kit dropdown opens, stays open, and items can be selected. Mouse input elsewhere in the plugin is unaffected.

<details>
<summary>Patch (against <code>b580a9f7</code>)</summary>

```diff
diff --git a/src/wine-host/editor.cpp b/src/wine-host/editor.cpp
index 835cd181..1705b6da 100644
--- a/src/wine-host/editor.cpp
+++ b/src/wine-host/editor.cpp
@@ -203,6 +203,85 @@ xcb_window_t get_x11_handle(HWND win32_handle) noexcept;
  */
 ATOM get_window_class() noexcept;
 
+namespace {
+
+bool popup_fix_tracing = false;
+int popup_fix_users = 0;
+HHOOK popup_fix_hook = nullptr;
+
+LRESULT CALLBACK popup_fix_cbt_proc(int code, WPARAM wParam, LPARAM lParam) {
+    if (code == HCBT_CREATEWND) {
+        const HWND window = reinterpret_cast<HWND>(wParam);
+        const LONG style = GetWindowLong(window, GWL_STYLE);
+        const LONG ex_style = GetWindowLong(window, GWL_EXSTYLE);
+
+        // Only captionless, non-resizable popup windows that are not supposed
+        // to show up in the task bar. Regular plugin windows and dialogs are
+        // left alone.
+        if (!(style & WS_CHILD) && (style & WS_POPUP) &&
+            (style & WS_CAPTION) != WS_CAPTION && !(style & WS_THICKFRAME) &&
+            (style & WS_SYSMENU) && !(ex_style & WS_EX_APPWINDOW)) {
+            SetWindowLong(window, GWL_STYLE, style & ~WS_SYSMENU);
+
+            // This goes to STDERR instead of through a `Logger` since the hook
+            // can outlive the editor that installed it. STDERR gets captured
+            // and logged by the plugin side.
+            if (popup_fix_tracing) {
+                std::cerr << "DEBUG: Removed WS_SYSMENU from popup window "
+                          << window << std::hex << " (style 0x" << style
+                          << ", exstyle 0x" << ex_style << std::dec
+                          << ") so Wine won't let the window manager manage it"
+                          << std::endl;
+            }
+        }
+    }
+
+    return CallNextHookEx(nullptr, code, wParam, lParam);
+}
+
+}  // namespace
+
+PopupStyleFixer::PopupStyleFixer(Logger& logger)
+    : active_(getenv("YABRIDGE_NO_POPUP_FIX") == nullptr) {
+    if (!active_) {
+        return;
+    }
+
+    if (popup_fix_users++ == 0) {
+        popup_fix_tracing = logger.editor_tracing_;
+        popup_fix_hook = SetWindowsHookEx(WH_CBT, popup_fix_cbt_proc, nullptr,
+                                          GetCurrentThreadId());
+    }
+}
+
+PopupStyleFixer::~PopupStyleFixer() noexcept {
+    if (!active_ || --popup_fix_users > 0) {
+        return;
+    }
+
+    if (popup_fix_hook) {
+        UnhookWindowsHookEx(popup_fix_hook);
+        popup_fix_hook = nullptr;
+    }
+}
+
+PopupStyleFixer::PopupStyleFixer(PopupStyleFixer&& other) noexcept
+    : active_(other.active_) {
+    other.active_ = false;
+}
+
+PopupStyleFixer& PopupStyleFixer::operator=(PopupStyleFixer&& other) noexcept {
+    if (this != &other) {
+        if (active_ && --popup_fix_users == 0 && popup_fix_hook) {
+            UnhookWindowsHookEx(popup_fix_hook);
+            popup_fix_hook = nullptr;
+        }
+        active_ = other.active_;
+        other.active_ = false;
+    }
+    return *this;
+}
+
 DeferredWin32Window::DeferredWin32Window(
     MainContext& main_context,
     std::shared_ptr<xcb_connection_t> x11_connection,
@@ -341,7 +420,8 @@ Editor::Editor(MainContext& main_context,
       host_window_(find_host_window(*x11_connection_,
                                     parent_window_,
                                     xcb_wm_window_role_property_)
-                       .value_or(parent_window_)) {
+                       .value_or(parent_window_)),
+      popup_style_fixer_(logger) {
     logger.log_editor_trace([&]() {
         return "DEBUG: host_window: " + std::to_string(host_window_);
     });
diff --git a/src/wine-host/editor.h b/src/wine-host/editor.h
index 1dc61551..fba7bc03 100644
--- a/src/wine-host/editor.h
+++ b/src/wine-host/editor.h
@@ -125,6 +125,32 @@ class DeferredWin32Window {
     std::shared_ptr<xcb_connection_t> x11_connection_;
 };
 
+/**
+ * Prevents Wine from handing plugin popup windows (like JUCE's popup menus) to
+ * the window manager, for as long as at least one instance exists. JUCE creates
+ * its popup menus with `WS_POPUP | WS_SYSMENU`, and Wine treats popup windows
+ * with a system menu as managed top level windows instead of override-redirect
+ * popups. Window managers like KWin then treat the menu as a regular window,
+ * which with Wine 10.x and later causes the menu to close again right after
+ * it opens. We strip `WS_SYSMENU` from captionless, non-resizable top level
+ * popup windows created on the GUI thread so Wine maps them as override-redirect
+ * windows, just like a native application's menus. This can be disabled by
+ * setting `YABRIDGE_NO_POPUP_FIX`.
+ */
+class PopupStyleFixer {
+   public:
+    explicit PopupStyleFixer(Logger& logger);
+    ~PopupStyleFixer() noexcept;
+
+    PopupStyleFixer(const PopupStyleFixer&) = delete;
+    PopupStyleFixer& operator=(const PopupStyleFixer&) = delete;
+    PopupStyleFixer(PopupStyleFixer&& other) noexcept;
+    PopupStyleFixer& operator=(PopupStyleFixer&& other) noexcept;
+
+   private:
+    bool active_ = false;
+};
+
 /**
  * TODO: This documentation needs to be updated with the recent changes to how
  *       embedding is handled.
@@ -442,4 +468,10 @@ class Editor {
      * `supports_ewmh_active_window()`.
      */
     mutable std::optional<bool> supports_ewmh_active_window_cache_;
+
+    /**
+     * Keeps plugin popup windows, like JUCE's popup menus, from being managed
+     * by the window manager while this editor is open.
+     */
+    PopupStyleFixer popup_style_fixer_;
 };
```

</details>

## Side finding: cursor offset when "Allow the window manager to control the windows" is off

While testing EZdrummer 3, the user hit a cursor offset that was exactly proportional to the plugin's distance from the top-left corner of the screen. It happens with upstream `master` too. The cause turned out to be that this Wine prefix had **"Allow the window manager to control the windows" disabled in `winecfg`** (`HKCU\Software\Wine\X11 Driver\Managed = N`). Turning it back on fixed it.

With managed mode off, Wine creates every window, including yabridge's editor window, as override-redirect. Then:

- no `MapRequest` is redirected to yabridge's wrapper window, and `WM_STATE` is never set;
- Wine ignores the synthetic `ConfigureNotify` events yabridge sends (`window_update_client_config()` returns early for `!data->managed`);
- Wine keeps assuming the editor is at its initial Win32 position (0,0).

This is easy to stumble into, since users often toggle that checkbox while trying to work around the dropdown bug. **A cheap improvement would be for yabridge to detect it**: after `Editor::show()`, if `wine_window_` has `override_redirect` set, or no `MapRequest` ever arrives, print a warning telling the user to enable that `winecfg` option.

## Credit

The root-cause analysis and the patch were done by **Claude, Anthropic's AI model, working in Claude Code**. It traced the problem through yabridge's editor code, Wine's `winex11.drv` and `win32u` sources and JUCE's `PopupMenu` implementation, then confirmed it with targeted debug builds that @singularity098 tested on real hardware.
