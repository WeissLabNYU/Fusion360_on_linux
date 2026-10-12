# Fusion360_on_linux

Claude's attempt to get Fusion360 running on Linux Ubuntu.

This repository documents a window-stacking defect in Wine's X11 driver that makes Autodesk
Fusion 360 stop responding to mouse clicks whenever a dialog with a grey backdrop is open,
explains why it happens, and gives the source change that fixes it, along with how to build and
verify that change yourself.

---

## Proven on

| | |
|---|---|
| OS | Ubuntu 26.04.1 LTS, GNOME, Wayland session (Fusion runs under XWayland); Ubuntu 24.04 LTS, GNOME, Xorg session |
| Fusion 360 | 2705.1.30 |
| Wine | 11.19 (Staging), WoW64 build |
| DXVK | 3.1.1 |

Nothing here is specific to a particular machine. The defect is in Wine's X11 driver and should
reproduce on any X11 or XWayland session whose window manager honours override-redirect, which
includes GNOME/Mutter and KDE/KWin.

The window-stacking fix documented below is **necessary but not sufficient**. Several other
problems have to be solved before Fusion runs at all, and two of them silently produce a
usable-looking application with a dead 3D viewport. See
[Other fixes required for a working install](#other-fixes-required-for-a-working-install).

---

## Symptom

Opening certain dialogs makes Fusion stop responding to the mouse. The dialog is drawn, but a
grey layer sits over it and every click, including **Cancel**, is swallowed. The application is
not frozen: it keeps redrawing and logging, and **Escape** still closes the dialog because the
keyboard focus is on the dialog. Four triggers were confirmed:

| trigger | dialog | dialog has an owner |
|---|---|---|
| "+" (new design) | What do you want to design? (845x546) | yes |
| Ctrl+S | Save (750x730) | no |
| Ctrl+O | Open (850x522) | no |
| File > Upload | Upload (750x503) | no |

Dialogs that Fusion shows *without* a grey layer (Export, Change Parameters, Preferences, the
sign-in session dialogs) were never affected.

Each grey layer is a **modal scrim**: a new window created for every dialog, sized to the main
window's client area, that dims the application and blocks input to it while the dialog is up.
Its Win32 properties are identical in all four cases: class `Qt683QWindowToolSaveBits`, style
`0x96000000` (WS_POPUP, no caption), extended style `0x00080080`
(WS_EX_TOOLWINDOW | WS_EX_LAYERED), owned by the main window, shown without activation.

Fusion's **menus have the same style bits** as the scrim, and the scrim's dialog is sometimes
owned and sometimes not. No rule based on style, class, owner, name or size can tell a scrim from
a menu. The fix therefore has to come from how Wine maps Win32 stacking onto X11, not from
recognising the scrim.

---

## What it is not

**Not a pointer or keyboard grab.** `XGrabKeyboard` and `XGrabPointer` both return `0`
(GrabSuccess) while the bug is active.

**Not the Autodesk cloud errors.** The log shows `GET 404 .../fhq/v1/spaces?type=FLC`,
`FHQService: failed to get FLC URN` and `PIM is disabled: no hub context` at the moment of
failure, and a WebView2 crash dump may exist. They are unrelated: the dialog is mapped, viewable
and healthy throughout.

**Not a missing XShape input region.** Wine 11.19 already gives `WS_EX_TRANSPARENT` windows an
empty input region (`sync_window_input_shape()`). The scrim is not transparent.

**Not fixable with a Wine virtual desktop.** Fusion crashes at launch inside
`wine explorer /desktop=...` with `DxgiFactory::CreateSwapChainForComposition: Not implemented`,
and DXVK is required for the 3D viewport.

**`Managed=N` works but regresses the desktop.** With
`HKCU\Software\Wine\AppDefaults\Fusion360.exe\X11 Driver  Managed=N` the defect disappears, but
so does every Fusion window from `_NET_CLIENT_LIST` (no taskbar entry, no alt-tab).

**Not a size rule.** An earlier version of this fix promoted any owned popup covering at least
90% of its owner. It fixed the "+" case but is a heuristic, and the reproducer below shows where
it fails: a half-size scrim, a scrim inserted *below* an existing dialog, and a side panel that
overlaps a dialog. It also leaves Fusion's side panels above the scrim while a dialog is open, so
they are not dimmed and still take clicks.

---

## Root cause: three defects in Wine's X11 driver

Wine creates each top-level window either **managed** (the window manager stacks it) or
**override-redirect** (Wine positions it and the window manager leaves it alone). X11 window
managers, Mutter included, keep override-redirect windows above all managed ones. Wine's X11
driver (`dlls/winex11.drv`) has three defects that together put the scrim above its own dialog.

1. **An override-redirect window cannot appear below a managed one, but Wine creates it anyway.**
   `is_window_managed()` makes a popup override-redirect when it is shown without activation, has
   no caption and owns no popups. That holds for menus and tooltips, which Win32 keeps on top. It
   is wrong for a window that has a managed window above it in Win32 order. The scrim is
   override-redirect while its dialog is managed, so X displays the scrim over the dialog. When
   the scrim was restacked below the dialog by hand, the window manager moved it back on top
   within 0.2 s without any request from Wine.
2. **Wine never asks the window manager for "below window X".** `sync_window_position()`
   only requests a raise, and only for the topmost Win32 window. A managed window that Win32
   inserts *under* another managed window keeps wherever the window manager put it.
3. **Wine infers Win32 Z-order from Expose events, which is invalid under a compositor.**
   `X11DRV_Expose()` moves an exposed window above everything that overlaps the exposed area
   (server request `update_window_zorder`). Under a compositing manager windows are redrawn as a
   whole, whatever their stacking, so the inference is wrong and corrupts the Win32 order. In the
   reproducer, a backdrop that the program placed *under* its dialog was moved above it within
   2 s. In Fusion, the scrim moved to the top of the Win32 order after a click.

## The fix

One rule per defect. There are no size, class, name or style rules.

| defect | change | rule |
|---|---|---|
| 1 | `is_below_managed_window()`, called from `is_window_managed()`; `make_lower_windows_managed()`, called from `X11DRV_WindowPosChanged()` | A window is managed if a visible managed window that overlaps it lies above it in Win32 Z-order. Whenever a managed window is shown, moved or restacked, the unmanaged windows below it that it overlaps are re-evaluated. Promotion is one-way, the only transition Wine supports. |
| 2 | `window_restack_below_prev()`, `restack_lower_managed()` | When Win32 places a managed window below another managed window, send the window manager an ICCCM ConfigureRequest with sibling + Below, using the nearest mapped managed window of the same process above it in Win32 order as the sibling. It is sent again once the window manager has mapped the window. Whenever a managed window is mapped or changes Z-order, the managed windows below it that it overlaps re-assert their position under it, so a window mapped later (such as a scrim) cannot end up below windows that Win32 keeps under it. Mutter honours this form; it ignores `_NET_RESTACK_WINDOW` from applications. |
| 3 | `X11DRV_Expose()` | Skip the Expose-based Z-order inference while a compositing manager owns `_NET_WM_CM_S<screen>`. The selection owner is re-checked on every Expose, so starting or stopping a compositor is followed. |

Menus, tooltips and drop-downs stay override-redirect, because Win32 keeps them above everything
and so no managed window is ever above them.

The patch is against wine-11.19 with the wine-staging v11.19 patchset applied, and touches two
files:

```diff
--- a/dlls/winex11.drv/event.c
+++ b/dlls/winex11.drv/event.c
@@ -829,6 +829,19 @@
 }
 
 
+/* is a compositing manager running? Expose events then say nothing about stacking */
+static BOOL compositing_manager_running( Display *display )
+{
+    static Atom cached_atom = None;
+    if (cached_atom == None)
+    {
+        char name[32];
+        snprintf( name, sizeof(name), "_NET_WM_CM_S%d", DefaultScreen( display ));
+        cached_atom = XInternAtom( display, name, False );
+    }
+    return XGetSelectionOwner( display, cached_atom ) != None;
+}
+
 /***********************************************************************
  *           X11DRV_Expose
  */
@@ -861,7 +874,7 @@
         OffsetRect( &rect, data->rects.visible.left - data->rects.client.left,
                     data->rects.visible.top - data->rects.client.top );
 
-    if (event->window != root_window)
+    if (event->window != root_window && !compositing_manager_running( event->display ))
     {
         abs_rect = rect;
         OffsetRect( &abs_rect, data->rects.client.left, data->rects.client.top );
@@ -874,7 +887,7 @@
         }
         SERVER_END_REQ;
     }
-    else flags &= ~RDW_ALLCHILDREN;
+    else if (event->window == root_window) flags &= ~RDW_ALLCHILDREN;
 
     release_win_data( data );
 
--- a/dlls/winex11.drv/window.c
+++ b/dlls/winex11.drv/window.c
@@ -387,6 +387,43 @@
     return ret;
 }
 
+static BOOL rects_intersect( const RECT *a, const RECT *b )
+{
+    return a->left < b->right && b->left < a->right && a->top < b->bottom && b->top < a->bottom;
+}
+
+/***********************************************************************
+ *              is_below_managed_window
+ *
+ * Check whether a visible managed window that intersects hwnd lies above it in the Win32
+ * Z-order. Such a window cannot be override-redirect: the X server and the window manager
+ * keep override-redirect windows above managed ones, so it would be displayed on top of a
+ * window that Win32 places over it.
+ */
+static BOOL is_below_managed_window( HWND hwnd )
+{
+    UINT i, dpi = NtUserGetDpiForWindow( hwnd );
+    RECT rect, other;
+    BOOL ret = FALSE;
+    HWND *list;
+
+    if (!NtUserGetWindowRect( hwnd, &rect, dpi ) || IsRectEmpty( &rect )) return FALSE;
+    if (!(list = build_hwnd_list())) return FALSE;
+
+    for (i = 0; list[i] != HWND_BOTTOM; i++)
+    {
+        if (list[i] == hwnd) break;
+        if (!(NtUserGetWindowLongW( list[i], GWL_STYLE ) & WS_VISIBLE)) continue;
+        if (!NtUserGetWindowRect( list[i], &other, dpi ) || !rects_intersect( &rect, &other )) continue;
+        if ((ret = is_managed( list[i] ))) break;
+    }
+
+    if (ret) TRACE( "%p %s is overlapped by managed window %p %s\n", hwnd, wine_dbgstr_rect( &rect ),
+                    list[i], wine_dbgstr_rect( &other ));
+    free( list );
+    return ret;
+}
+
 /* returns the HWND for the X11 window, or the desktop window if it isn't a Wine window */
 static HWND hwnd_from_window( Display *display, Window window )
 {
@@ -463,6 +500,9 @@
     if (ex_style & WS_EX_APPWINDOW) return TRUE;
     /* windows that own popups are managed */
     if (has_owned_popups( hwnd )) return TRUE;
+    /* windows overlapped from above by a managed window are managed, an override-redirect
+     * window would be displayed above it */
+    if (is_below_managed_window( hwnd )) return TRUE;
     /* default: not managed */
     return FALSE;
 }
@@ -1870,6 +1910,66 @@
     return TRUE;
 }
 
+/***********************************************************************
+ *              window_restack_below_prev
+ *
+ * Ask the window manager to stack a managed window directly below its nearest visible
+ * Win32 predecessor, when that predecessor is managed too.
+ */
+static void window_restack_below_prev( struct x11drv_win_data *data )
+{
+    struct x11drv_win_data *prev_data;
+    XWindowChanges changes = {0};
+    HWND prev = data->hwnd;
+
+    if (!data->managed || !data->whole_window || data->current_state.wm_state != NormalState) return;
+    while (!changes.sibling && (prev = NtUserGetWindowRelative( prev, GW_HWNDPREV )))
+    {
+        if (!(NtUserGetWindowLongW( prev, GWL_STYLE ) & WS_VISIBLE)) continue;
+        if (!(prev_data = get_win_data( prev ))) continue;
+        if (prev_data->managed && prev_data->whole_window && prev_data->current_state.wm_state == NormalState)
+            changes.sibling = prev_data->whole_window;
+        release_win_data( prev_data );
+    }
+    if (!changes.sibling) return;
+
+    changes.stack_mode = Below;
+    TRACE( "window %p/%lx, requesting stacking below %p/%lx\n", data->hwnd, data->whole_window, prev, changes.sibling );
+    XReconfigureWMWindow( data->display, data->whole_window, data->vis.screen, CWSibling | CWStackMode, &changes );
+}
+
+/***********************************************************************
+ *              restack_lower_managed
+ *
+ * hwnd is a managed window that was just mapped or changed Z-order. Re-assert the stacking of
+ * the managed windows below it in Win32 order that it overlaps, so that a window inserted above
+ * existing managed windows is also above them on screen.
+ */
+static void restack_lower_managed( HWND hwnd )
+{
+    UINT i, dpi = NtUserGetDpiForWindow( hwnd );
+    BOOL below = FALSE;
+    RECT rect, other;
+    HWND *list;
+
+    if (!NtUserGetWindowRect( hwnd, &rect, dpi ) || IsRectEmpty( &rect )) return;
+    if (!(list = build_hwnd_list())) return;
+
+    for (i = 0; list[i] != HWND_BOTTOM; i++)
+    {
+        struct x11drv_win_data *data;
+
+        if (list[i] == hwnd) { below = TRUE; continue; }
+        if (!below) continue;
+        if (!(NtUserGetWindowLongW( list[i], GWL_STYLE ) & WS_VISIBLE)) continue;
+        if (!NtUserGetWindowRect( list[i], &other, dpi ) || !rects_intersect( &rect, &other )) continue;
+        if (!(data = get_win_data( list[i] ))) continue;
+        window_restack_below_prev( data );
+        release_win_data( data );
+    }
+    free( list );
+}
+
 void window_wm_state_notify( struct x11drv_win_data *data, unsigned long serial, UINT value, Time time )
 {
     UINT *desired = &data->desired_state.wm_state, *pending = &data->pending_state.wm_state, *current = &data->current_state.wm_state;
@@ -1891,6 +1991,11 @@
     /* send any pending changes from the desired state */
     window_request_desired_state( data );
 
+    if (data->current_state.wm_state == NormalState)
+    {
+        window_restack_below_prev( data );
+        restack_lower_managed( data->hwnd );
+    }
     if (data->current_state.wm_state == NormalState) NtUserSetProp( data->hwnd, focus_time_prop, (HANDLE)time );
     else if (!data->wm_state_serial) NtUserRemoveProp( data->hwnd, focus_time_prop );
 }
@@ -2169,6 +2274,11 @@
                                         window_rect.top - old_rects->window.top );
 
     window_set_config( data, new_rect, above );
+    if (!above && (!(swp_flags & SWP_NOZORDER) || (swp_flags & SWP_SHOWWINDOW)))
+    {
+        window_restack_below_prev( data );
+        restack_lower_managed( data->hwnd );
+    }
 }
 
 
@@ -3260,6 +3370,48 @@
 
 
 /***********************************************************************
+ *              make_lower_windows_managed
+ *
+ * hwnd is a visible managed window. Re-evaluate every visible unmanaged window below it that
+ * it intersects, so that is_window_managed() can promote it (see is_below_managed_window).
+ */
+static void make_lower_windows_managed( HWND hwnd )
+{
+    static const UINT flags = SWP_NOACTIVATE | SWP_NOZORDER | SWP_NOSIZE | SWP_NOMOVE | SWP_NOREDRAW |
+                              SWP_DEFERERASE | SWP_NOSENDCHANGING | SWP_STATECHANGED;
+    UINT i, count = 0, dpi = NtUserGetDpiForWindow( hwnd );
+    HWND *list, *lower;
+    RECT rect, other;
+    BOOL below = FALSE;
+
+    if (!NtUserGetWindowRect( hwnd, &rect, dpi ) || IsRectEmpty( &rect )) return;
+    if (!(list = build_hwnd_list())) return;
+    for (i = 0; list[i] != HWND_BOTTOM; i++) ;
+    if (!(lower = malloc( (i + 1) * sizeof(*lower) ))) { free( list ); return; }
+
+    for (i = 0; list[i] != HWND_BOTTOM; i++)
+    {
+        struct x11drv_win_data *data;
+
+        if (list[i] == hwnd) { below = TRUE; continue; }
+        if (!below) continue;
+        if (!(NtUserGetWindowLongW( list[i], GWL_STYLE ) & WS_VISIBLE)) continue;
+        if (!NtUserGetWindowRect( list[i], &other, dpi ) || !rects_intersect( &rect, &other )) continue;
+        if (!(data = get_win_data( list[i] ))) continue;
+        if (data->whole_window && !data->managed && !data->embedded) lower[count++] = list[i];
+        release_win_data( data );
+    }
+    free( list );
+
+    for (i = 0; i < count; i++)
+    {
+        TRACE( "re-evaluating %p, below managed window %p\n", lower[i], hwnd );
+        NtUserSetWindowPos( lower[i], 0, 0, 0, 0, 0, flags );
+    }
+    free( lower );
+}
+
+/***********************************************************************
  *		WindowPosChanged   (X11DRV.@)
  */
 void X11DRV_WindowPosChanged( HWND hwnd, HWND insert_after, HWND owner_hint, UINT swp_flags,
@@ -3339,8 +3491,13 @@
     was_fullscreen &= data->desired_state.wm_state != NormalState;
 
     XFlush( data->display );  /* make sure changes are done before we start painting again */
+    is_managed = data->managed && (new_style & WS_VISIBLE);
     release_win_data( data );
 
+    if (is_managed && (swp_flags & (SWP_NOZORDER | SWP_NOMOVE | SWP_NOSIZE | SWP_SHOWWINDOW)) !=
+                      (SWP_NOZORDER | SWP_NOMOVE | SWP_NOSIZE))
+        make_lower_windows_managed( hwnd );
+
     if (was_fullscreen) NtUserClipCursor( NULL );
 }
 
```

### Behaviour changes to expect

- **Fusion's side panels become managed windows** once a managed window has been above them,
  for example during the first dialog. They are flagged skip-taskbar and carry no decorations,
  and they are now correctly dimmed under a dialog's scrim instead of floating above it.
- **Clicking inside a panel can draw the main title bar as inactive.** That is the actual Win32
  state: clicking a tree item makes the panel the foreground window (measured with
  `GetForegroundWindow()`). Stock Wine hides this because the window manager never sees the
  unmanaged panel take focus. Shortcuts such as Ctrl+S still work from the panel.

---

## Building a patched Wine

The patch touches two files, but Wine must be built as a whole.

### Prerequisites

A PE cross-compiler is mandatory; `configure` aborts with
`i386 PE compiler not found` without it.

```bash
sudo apt install -y build-essential flex bison gcc-mingw-w64-x86-64 gcc-mingw-w64-i686 \
  libgnutls28-dev libunwind-dev libosmesa6-dev libfreetype-dev libx11-dev libxext-dev \
  libxcomposite-dev libxcursor-dev libxfixes-dev libxi-dev libxrandr-dev libxrender-dev \
  libxinerama-dev libxkbcommon-dev libxkbregistry-dev libwayland-dev libvulkan-dev \
  libasound2-dev libpulse-dev libdbus-1-dev libfontconfig-dev libgl-dev libudev-dev \
  libcups2-dev libkrb5-dev libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev \
  libpcap-dev libusb-1.0-0-dev libv4l-dev libsdl2-dev
```

### Why Staging is required

Build **wine-staging**, not vanilla Wine. Staging carries the `ntdll-Hide_Wine_Exports`
patchset, which makes the registry value `HKCU\Software\Wine  HideWineExports = Y` work. Without
it Fusion logs `Fusion is running on GNU/Linux using WINE` and takes different code paths.

### Build

```bash
mkdir -p "$HOME/wine-build" && cd "$HOME/wine-build"

git clone --depth 1 --branch wine-11.19 https://github.com/wine-mirror/wine.git wine
git clone --depth 1 --branch v11.19 https://github.com/wine-staging/wine-staging.git

python3 wine-staging/staging/patchinstall.py --all --destdir="$PWD/wine"

# save the diff above as winex11-zorder.patch, then:
cd wine && git apply --check ../winex11-zorder.patch && git apply ../winex11-zorder.patch && cd ..

mkdir build && cd build
../wine/configure --enable-archs=i386,x86_64 \
                  --prefix="$HOME/wine-build/install" \
                  --disable-tests
make -j"$(nproc)"
make install
```

After the first full build, changes to the X11 driver rebuild in seconds. Only
`lib/wine/x86_64-unix/winex11.so` differs between a stock and a patched build, so keep both
copies and swap that one file to compare them.

### A note on glibc, if you build and run on different systems

Binaries linked against an older glibc run on a newer one, but not the reverse. Build on
whichever system has the **older** glibc, and check with `ldd <install>/bin/wine | grep 'not found'`.

### Install alongside, not over

Keep the stock build and select between them at launch, so a regression is one environment
variable away from being undone:

```bash
BASE="$HOME/fusion-standalone"
export WINEPREFIX="$BASE/prefix"
WINE_DIR="${FUSION_WINE:-wine11-patched}"
WINE="$BASE/$WINE_DIR/bin/wine"
export WINEDEBUG="${WINEDEBUG:--all}"
FEXE=$(find "$WINEPREFIX/drive_c" -name Fusion360.exe 2>/dev/null | head -1)
exec "$WINE" "$FEXE" "$@"
```

```bash
FUSION_WINE=wine11 fusion360-wine    # roll back to the stock build
```

---

## Verifying the fix without launching Fusion

Do not debug this by starting Fusion and clicking. Reproduce the window arrangements with a small
Win32 program; each run takes seconds.

`scrimtest2.c` creates a main window, a scrim and a dialog in one of eight layouts, selected by
the first command-line letter, and logs its own Win32 Z-order to `Z:\tmp\stz_<mode>.txt`:

```c
/* scrimtest2.c - reproduce Fusion's modal-scrim arrangement under Wine.
 * usage: scrimtest2.exe <mode>
 *   A  scrim (plain WS_POPUP) shown first, then dialog activated      (old reproducer)
 *   B  dialog first, then scrim inserted BELOW the dialog in Z-order    (SetWindowPos(scrim, dlg))
 *   C  as A, scrim WS_EX_TOOLWINDOW|WS_EX_LAYERED alpha 128             (Fusion's real exstyle)
 *   D  as B with C's exstyle
 *   E  small owned popup (300x400 panel, TOOLWINDOW) + dialog          (no scrim; panel vs dialog)
 *   F  scrim the SAME size as owner (no oversize), TOOLWINDOW|LAYERED, order A
 *   G  scrim at 50% of the owner, TOOLWINDOW|LAYERED, order A          (defeats any size rule)
 *   H  small owned TOOLWINDOW panel that the dialog overlaps, order A  (panel must end up below)
 */
#include <windows.h>
#include <stdio.h>
static FILE *zlog;
static void dumpz( const char *tag )
{
    HWND h = GetTopWindow( NULL ); WCHAR t[64]; int n = 0;
    if (!zlog) return;
    fprintf( zlog, "%-14s:", tag );
    for (; h; h = GetWindow( h, GW_HWNDNEXT ))
    {
        if (!IsWindowVisible( h )) continue;
        GetWindowTextW( h, t, 63 );
        if (wcsncmp( t, L"scrimtest-", 10 )) continue;
        fprintf( zlog, " %ls", t + 10 ); n++;
    }
    fprintf( zlog, "\n" ); fflush( zlog );
}
static LRESULT CALLBACK wp( HWND h, UINT m, WPARAM w, LPARAM l )
{
    if (m == WM_DESTROY) { PostQuitMessage(0); return 0; }
    if (m == WM_ERASEBKGND) { RECT r; GetClientRect( h, &r ); FillRect( (HDC)w, &r, (HBRUSH)GetWindowLongPtrW( h, GWLP_USERDATA ) ); return 1; }
    return DefWindowProcW( h, m, w, l );
}
int WINAPI wWinMain( HINSTANCE inst, HINSTANCE p, WCHAR *cmd, int show )
{
    WNDCLASSW wc = {0}; MSG msg; HWND main_w, scrim, dlg; RECT mr; WCHAR mode = cmd && cmd[0] ? cmd[0] : L'A';
    DWORD sex = (mode == L'C' || mode == L'D' || mode == L'E' || mode == L'F' || mode == L'G') ? (WS_EX_TOOLWINDOW | WS_EX_LAYERED) : 0;
    int w, h, x, y;
    if (mode == L'E' || mode == L'H') sex = WS_EX_TOOLWINDOW;
    wc.lpfnWndProc = wp; wc.hInstance = inst; wc.hCursor = LoadCursorW( NULL, (const WCHAR *)IDC_ARROW ); wc.lpszClassName = L"scrimtest";
    RegisterClassW( &wc );
    { WCHAR fn[64]; swprintf( fn, 64, L"Z:\\tmp\\stz_%c.txt", mode ); zlog = _wfopen( fn, L"w" ); }
    main_w = CreateWindowExW( 0, L"scrimtest", L"scrimtest-main", WS_OVERLAPPEDWINDOW, 200, 150, 1200, 800, NULL, NULL, inst, NULL );
    SetWindowLongPtrW( main_w, GWLP_USERDATA, (LONG_PTR)GetStockObject( GRAY_BRUSH ) );
    ShowWindow( main_w, SW_SHOW ); UpdateWindow( main_w ); GetWindowRect( main_w, &mr );
    w = mr.right - mr.left; h = mr.bottom - mr.top; x = mr.left; y = mr.top;
    if (mode == L'A' || mode == L'B' || mode == L'C' || mode == L'D') { y -= 30; h += 30; }
    if (mode == L'E') { x += 20; y += 120; w = 300; h = 400; }
    if (mode == L'H') { x += 500; y += 300; w = 300; h = 400; }
    if (mode == L'G') { x += w / 4; y += h / 4; w /= 2; h /= 2; }
    scrim = CreateWindowExW( sex, L"scrimtest", L"scrimtest-scrim", WS_POPUP, x, y, w, h, main_w, NULL, inst, NULL );
    SetWindowLongPtrW( scrim, GWLP_USERDATA, (LONG_PTR)GetStockObject( DKGRAY_BRUSH ) );
    if (sex & WS_EX_LAYERED) SetLayeredWindowAttributes( scrim, 0, 128, LWA_ALPHA );
    dlg = CreateWindowExW( WS_EX_DLGMODALFRAME, L"scrimtest", L"scrimtest-dialog", WS_POPUP | WS_CAPTION | WS_SYSMENU,
                           mr.left + 400, mr.top + 250, 600, 400, main_w, NULL, inst, NULL );
    SetWindowLongPtrW( dlg, GWLP_USERDATA, (LONG_PTR)GetStockObject( WHITE_BRUSH ) );
    if (mode == L'B' || mode == L'D')
    {
        ShowWindow( dlg, SW_SHOW ); SetForegroundWindow( dlg );
        Sleep( 500 );
        dumpz( "dialog shown" );
        SetWindowPos( scrim, dlg, 0, 0, 0, 0, SWP_NOMOVE | SWP_NOSIZE | SWP_NOACTIVATE | SWP_SHOWWINDOW );
        dumpz( "scrim inserted" );
    }
    else
    {
        ShowWindow( scrim, SW_SHOWNOACTIVATE );
        dumpz( "scrim shown" );
        Sleep( 300 );
        ShowWindow( dlg, SW_SHOW );
        dumpz( "dialog shown" );
        SetForegroundWindow( dlg );
        dumpz( "dialog fg" );
    }
    SetTimer( NULL, 1, 2000, NULL );
    while (GetMessageW( &msg, NULL, 0, 0 ) > 0) { if (msg.message == WM_TIMER && !msg.hwnd) { dumpz( "after 2s" ); KillTimer( NULL, 1 ); } TranslateMessage( &msg ); DispatchMessageW( &msg ); }
    return 0;
}
```

```bash
x86_64-w64-mingw32-gcc -municode -mwindows -O1 -o scrimtest2.exe scrimtest2.c -lgdi32 -luser32
```

Run each mode (`wine scrimtest2.exe A` ... `H`) for about 5 s, then compare the Win32 order in the
log with the X stacking order of the same windows. `XQueryTree` on the root lists top-level
windows bottom to top; managed windows are reparented into a frame, so look one level down too.

### Results

Same Wine tree each time; only `winex11.so` swapped. A mode passes when the dialog is above the
scrim on screen and the X order matches the Win32 order.

| driver | modes passing |
|---|---|
| stock 11.19-staging | 0/8 |
| 90% size rule (previous fix) | 3/8: fails G (half-size scrim), B and D (scrim inserted below its dialog), H (overlapping panel) |
| defect 1 fixed | 6/8 |
| defects 1 + 2 fixed | 7/8 |
| **defects 1 + 2 + 3 fixed (this patch)** | **8/8**, and the Win32 order stays as the program set it |

A second reproducer stepped one window set through 16 states: show and close a modal dialog
three times, show and hide a reused popup twice, move the main window off the left edge and back,
minimise and restore. With this patch, X stacking matched Win32 order in all 16 states. With the
stock driver, the scrim was above its dialog in every modal state, and the docked panel was below
the main window in Win32 order in every state where it was visible. Window rectangles were
identical under both drivers.

### Wine's own conformance tests

`user32:win`, `user32:msg`, `user32:input` and `user32:menu` were run on GNOME/Mutter (Xorg) with
each driver. The test program comes from a second, test-enabled build directory:
`make dlls/user32/tests/x86_64-windows/user32_test.exe`.

| test | stock | this patch |
|---|---|---|
| `win` | 3 to 5 failed checks in every run (14 runs); the 3 topmost checks (`win.c:12318`, `12326`, `12332`) fail every time | 6 runs: 0 failures in 5; 1 run with 2 failures (`win.c:10776`, `10777`), two checks that also fail in some stock runs. The 3 topmost checks pass |
| `msg` | 7 failed checks | the same 7 |
| `input`, 4 runs each | 39 to 40 failures | 39 to 40, the same checks; `input.c:6776` is flaky under both drivers |
| `menu` | 0 failures | 0 failures |

`user32:dialog` times out on this desktop with both drivers.

### Fusion itself

Fusion 360 2705.1.30, signed in and online, on Ubuntu 24.04 (GNOME, Xorg) and Ubuntu 26.04.1
(GNOME, Wayland/XWayland):

- **"+", Save, Open, Upload** (Xorg): the dialog is above the scrim, Cancel closes it, and the
  X stacking order is main window < side panels < scrim < dialog, all managed.
- **Save and Open** (Wayland session): the same stacking order. With the previous 90% fix, the
  side panels were override-redirect and stacked *above* both the scrim and the dialog, so they
  stayed clickable while a modal dialog was open.
- **Window operations**: maximize, minimize and restore, unmaximize, resize and move were driven
  from the window manager. In normal launches the Win32 state (minimized flag, window rectangle)
  follows the window manager and the main window answers a right-click with its marking menu
  afterwards. See [Known issue](#known-issue-fusion-stops-following-the-window-manager) for an
  intermittent failure that occurs with or without this patch.
- The main window stays in `_NET_CLIENT_LIST`; promoted panels are skip-taskbar.
- A title bar that is missing or stale after start-up, unmaximize or a move occurs identically with
  the stock driver. It is not caused by this patch.

### Testing traps

- **Rootless XWayland drops XTEST pointer input.** On a GNOME Wayland session, `xdotool
  mousemove`/`click` without `--window` does not reach X clients, and `XQueryPointer` reports the
  root window even when a window is under the pointer. Use window-targeted events
  (`xdotool click --window <id>`) for scripting. These bypass X input routing, so a stacking fix
  still has to be checked by reading the stacking order or by clicking with a real mouse.
- **Mutter keeps minimized windows mapped.** After `xdotool windowminimize`, `xwininfo` still
  reports `IsViewable` with the old geometry. The minimize has still happened:
  `_NET_WM_STATE_HIDDEN` is set, and Win32 reports `WS_MINIMIZE` with the rectangle
  `-32000,-32000 160x31`. Check those, not the map state.
- **A maximized window ignores `xdotool windowsize` and `windowmove`.** Unmaximize first
  (`wmctrl -r <win> -b remove,maximized_vert,maximized_horz`) and check that the geometry changed.
- **Frame windows share the title.** On GNOME, `xdotool search --name` also returns the
  window-manager frame (`WM_CLASS` `mutter-x11-frames`), which is larger than the client. Filter
  on `WM_CLASS` `fusion360.exe`.

### Known issue: Fusion stops following the window manager

This is not fixed by the patch, and it is not caused by it. It occurs with the stock driver too.

**What you see.** After you maximize Fusion from the window manager (double-click on the title bar,
Super+Up, `wmctrl -b add,maximized_vert,maximized_horz`), the window stops reacting to the mouse.
It is still drawn, but right-clicks get no marking menu, a minimize from the window manager is
never seen by Fusion, and the window stays at its maximized Win32 rectangle whatever the window
manager does with it. The right-hand palette may be drawn as a blank white rectangle beside the
window. Restarting Fusion is the only recovery found so far.

**How often.** Fusion was launched repeatedly on Ubuntu 24.04 (GNOME, Xorg) and maximized,
minimized and restored from the window manager. The failure occurred in 4 of 19 launches with the
stock driver and in 2 of 21 with this patch, always at the first maximize. The difference is
within chance. Launches that did not fail at the first maximize did not fail later in the test.

**What is known about the cause.**

- Wine receives the window manager's changes (`WM_STATE`, `_NET_WM_STATE`, ConfigureNotify) and
  logs them as unexpected, but never passes them on to Win32.
- In a traced failing launch, the Win32 maximize started and the main window and its side panels
  were resized. Fusion's GUI thread then stopped before the right-hand palette's content was
  resized. Wine's per-window state lock, taken while Win32 applies a window-manager change, was
  never released, and the thread processed no further posted or input messages for its windows.
- The thread is not hung: it answers `SendMessageTimeout(WM_NULL)` within 25 ms and
  `IsHungAppWindow()` returns false. A posted `WM_NULL` does not revive it, and neither does a
  minimize/restore.

**Workaround.** Leave the Fusion window at one size, maximized or not, and avoid toggling maximize
from the window manager. Save often; Fusion's document recovery covers a forced restart.

---

## Emergency recovery, without the patch

On a stock build, **Escape** closes the stuck dialog, because keyboard focus is on the dialog.
Alternatively, clear the scrim's X input region so clicks pass through to the dialog. This is a
stopgap: modal blocking is lost and it has to be repeated for each new dialog.

```python
# clear the input region of the full-size override-redirect child
import ctypes, ctypes.util
xext = ctypes.CDLL(ctypes.util.find_library("Xext"))
x11 = ctypes.CDLL(ctypes.util.find_library("X11"))
x11.XOpenDisplay.restype = ctypes.c_void_p
dpy = x11.XOpenDisplay(None)
xext.XShapeCombineRectangles(ctypes.c_void_p(dpy), ctypes.c_ulong(WINDOW_ID),
                             ctypes.c_int(2),   # ShapeInput
                             0, 0, None, 0, ctypes.c_int(0), 0)
x11.XSync(ctypes.c_void_p(dpy), 0)
```

`WINDOW_ID` is the viewable override-redirect window sized to the main window's client area.

---

## Other fixes required for a working install

The stacking defect above is the only change to Wine's *source*. It is not the only change
required. Everything in this section must also be in place, and the two rendering items in
particular fail in a way that looks like a working installation.

### Wine version

| Wine | Result |
|---|---|
| 9.22 | Null-pointer crash inside `qwindows.dll`; no window ever appears |
| 10.11-staging | Window appears, then aborts on unimplemented `bcp47langs.dll.GetUserLanguages` during sign-in |
| **11.19-staging** | Reaches sign-in cleanly; the `bcp47langs` abort is gone and module errors drop from dozens to 4 |

Use the **WoW64** build. A plain `-amd64` build exits immediately with
`/lib/ld-linux.so.2: could not open` on any system without i386 multiarch.

Set the prefix to **Windows 11** mode.

### 3D rendering, part 1: DXVK is mandatory

With Wine's default wined3d (Direct3D → OpenGL) path, the application starts, the UI chrome
draws correctly, and **the 3D viewport is blank white**. Install DXVK so Direct3D 11 is
translated to Vulkan instead:

```
[HKEY_CURRENT_USER\Software\Wine\DllOverrides]
"d3d11"="native"
"dxgi"="native"
"d3d10core"="native"
"d3d9"="native"
```

### 3D rendering, part 2: a native `d3dcompiler_47` is also mandatory

DXVK alone is not enough, and this is the most deceptive failure of the whole exercise. With
DXVK installed and reporting no errors, **sketches and edges draw but solid surfaces never
rasterise**. Wine's builtin `d3dcompiler_47` cannot compile the application's HLSL shaders at
runtime.

Objective evidence — pixel statistics of the same 50 mm cube, rendered through the application's
own API in four visual styles. A single unique colour means nothing was drawn:

| visual style | before | after |
|---|---|---|
| Shaded | 1 colour, stdev 0.00 | **199 colours, stdev 72.12** |
| Shaded + hidden edges | 53 colours, stdev 1.56 | **677 colours, stdev 73.13** |
| Shaded, visible edges only | 1 colour, stdev 0.00 | 166 colours, stdev 73.04 |

The geometry was correct the whole time — the cube measured exactly 125 cm³ — so this cannot be
caught by checking that modelling operations succeed. **Only a rendered pixel tells you.** Any
automated test of a CAD application under Wine should assert on image statistics, not just on
API return values.

A genuine `d3dcompiler_47.dll` is usually **already inside the prefix**, shipped with the
WebView2 runtime that the installer pulls in:

```
drive_c/Program Files (x86)/Microsoft/EdgeWebView/Application/<version>/d3dcompiler_47.dll
```

Back up the builtin, copy that file into `system32`, and override:

```
[HKEY_CURRENT_USER\Software\Wine\DllOverrides]
"d3dcompiler_47"="native,builtin"
```

### Hiding Wine from the application

By default the application logs `Fusion is running on GNU/Linux using WINE` and takes different
code paths accordingly. Wine-staging can hide its exports:

```
[HKEY_CURRENT_USER\Software\Wine]
"HideWineExports"="Y"
```

With this set, the application's embedded Python reports `os.name == 'nt'` as it would on
Windows. This is a **staging-only** feature — another reason to build staging rather than
vanilla Wine.

### Qt PDF plugin crash (older Wine)

On Wine 9.22 a second crash masks the first. The Qt image-format plugin
`plugins/imageformats/qpdf.dll` imports:

```
Qt6Core.dll
Qt6Gui.dll
Qt6Pdf.dll      <- never shipped by the vendor
```

The load fails, returns NULL, and the result is dereferenced. Rename the plugin to
`qpdf.dll.disabled`. Not required on 11.19, but worth knowing: while it is present it hides
whatever else is wrong.

### Sign-in returns to the software store instead of the application

Browser-based sign-in completes by invoking an **`adskidmgr://`** URL. The browser runs on
Linux, the application runs inside Wine, and nothing on the desktop handles that scheme — so
the desktop offers to search the software store and the login never returns.

Register a handler that passes the URL into the prefix:

1. A small script that runs `AdskIdentityManager.exe` inside the prefix with `"$1"`.
2. A `.desktop` entry for it containing `MimeType=x-scheme-handler/adskidmgr;`.
3. `xdg-mime default <entry>.desktop x-scheme-handler/adskidmgr`

Verify with `xdg-mime query default x-scheme-handler/adskidmgr`.

### Sign-in race on restart

Restarting too quickly produces:

```
IDSDKAuth: idsdk_set_product_config failed with error IDSDK_E_SERVER_PROCESS_NOT_READY
IDSDKAuth: Showing blocking Quit dialog. title=Unable to sign in
```

The Identity SDK helper has not finished shutting down. A reliable sequence is: kill the
application and `AdskIdentityManager`, wait ~8 s, then `wineserver -k`, then wait ~10 s more
before relaunching.

Do **not** delete `AppData/Local/Autodesk/Web Services` while troubleshooting — it holds the
authentication token, and removing it forces a full re-login.

### No icon in the taskbar

Two separate causes:

- Pointing `Icon=` at a Windows `.ico` does not resolve. Extract the embedded 256x256 PNG from
  the application's `.ico` into the hicolor theme and reference it by name.
- Without `StartupWMClass`, a running window has nothing tying it back to the launcher. Read the
  real class from a live window with `xprop WM_CLASS` — it is `fusion360.exe` — and set
  `StartupWMClass=fusion360.exe` in the `.desktop` entry.

### DLL overrides to leave alone

`mfc140u` and `msvcp140` must stay **native**. Wine ships no builtin `mfc140u`, so forcing
builtin there breaks startup.

---

## Status

Not yet submitted upstream to WineHQ. The three changes are independent and can be reviewed
separately; the conformance results above are the evidence to start from. Contributions are welcome.
