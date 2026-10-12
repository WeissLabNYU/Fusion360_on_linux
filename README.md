# Fusion360_on_linux

Claude's attempt to get Fusion360 running on Linux Ubuntu.

This repository documents a window-manager defect in Wine that makes Autodesk Fusion 360 stop
responding to mouse clicks, explains why it happens, and gives the source change that fixes it
along with how to build and verify that change yourself.

---

## Proven on

| | |
|---|---|
| OS | Ubuntu 26.04.1 LTS, GNOME, Wayland session (Fusion runs under XWayland) |
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

Clicking the **"+" (new design)** button greys it out, and from that moment the application stops
responding to clicks entirely. Dialogs and popups opened afterwards do not respond either. The
application is not frozen — it continues to redraw, animate and log normally — only mouse input
is lost. Restarting Fusion is the only recovery.

---

## What it is not

Each of these was investigated and ruled out. They are listed because they are plausible, they
cost time, and two of them produce log output that looks damning but is coincidental.

**Not a pointer or keyboard grab.** `XGrabKeyboard` and `XGrabPointer` both return `0`
(GrabSuccess), meaning no other client held a grab. The screen is not locked.

**Not the Autodesk cloud errors.** The log contains, at the moment of failure:

```
E [pim.core] PIM is disabled: no hub context
E GET 404 https://api.aps.usa.autodesk.com/fhq/v1/spaces?type=FLC&fusionId=<redacted>
          workflow:getFLCSpace
W FHQService: failed to get FLC URN, response code:
```

together with a WebView2 crash dump under `AppData/Local/ADPWebView/`. These are unrelated. A
window inventory taken *while the bug is active* shows the `845x546` "What do you want to design?"
dialog present, mapped and viewable. Nothing failed to open and nothing failed to tear down.

**Not a missing XShape input region.** A reasonable theory is that the covering window is
`WS_EX_TRANSPARENT` and Wine forgot to give it an empty X input region. Wine 11.19 already
implements precisely that, in `sync_window_input_shape()` (`dlls/winex11.drv/window.c`), called
from both `sync_window_style()` and window creation. The covering window is simply not transparent.

**Not fixable with a Wine virtual desktop.** Running Fusion inside `wine explorer /desktop=...`
is the textbook remedy for override-redirect stacking problems, because it makes Wine arbitrate
Z-order with Windows semantics. It cannot be used here — Fusion crashes at launch with:

```
err: DxgiFactory::CreateSwapChainForComposition: Not implemented
```

DXVK does not implement composition swapchains, which is the path Fusion/WebView2 takes when
drawing into a virtual desktop. DXVK is required for the 3D viewport to rasterise surfaces at
all, so virtual desktop and working 3D are mutually exclusive.

**`Managed=N` works but regresses the desktop.** Setting

```
HKCU\Software\Wine\AppDefaults\Fusion360.exe\X11 Driver   Managed = N
```

does eliminate the defect, by putting every Fusion window under Wine's own Z-order so the two
regimes described below cannot disagree. But it also removes every Fusion window from
`_NET_CLIENT_LIST`: no taskbar entry, no alt-tab, no dash icon. Not recommended.

---

## Root cause

Fusion's modal dialogs are drawn over a **scrim** — a full-size backdrop window that dims the
application and blocks input to the main window while the dialog is up. That is correct and
intended behaviour; a modal scrim is *supposed* to swallow clicks aimed at the window behind it.

The defect is that Wine places the scrim and its dialog into two different stacking regimes.

In `dlls/winex11.drv/window.c`, `set_style_hints()` advertises the scrim to the window manager
as a dialog:

```c
/* Only use dialog type for owned popups. ... */
if (((style & WS_POPUP) || (ex_style & WS_EX_DLGMODALFRAME)) && owner)
    window_set_net_wm_window_type( data, XATOM__NET_WM_WINDOW_TYPE_DIALOG );
```

and sets `WM_TRANSIENT_FOR` pointing at the owner. Both are hints that **only a window manager
acts on**.

But `is_window_managed()` in the same file leaves that window **unmanaged** — created with
`override_redirect = True`, which tells the window manager not to touch it. A window is left
unmanaged when it is shown with `SWP_NOACTIVATE`, has no `WS_CAPTION` and no `WS_THICKFRAME`,
is not a popup with `WS_SYSMENU`, is not full-screen, lacks `WS_EX_APPWINDOW` and owns no popups.
A modal scrim matches every one of those conditions. In particular it is **not** full-screen: it
is sized to its owner window, not to the monitor, so the `fullscreen` escape never fires.

So Wine simultaneously tells the window manager "this is a transient dialog, please manage its
stacking" and "do not manage this window". The result:

| window | regime | who controls Z-order |
|---|---|---|
| the scrim | override-redirect | Wine, on the Win32 side |
| the dialog | WM-managed | the window manager |

Nothing reconciles the two, and X stacks override-redirect windows above managed ones. The scrim
ends up above the very dialog it exists to sit behind, and absorbs every click aimed at it. The
dialog is alive and functional the whole time — it is simply buried.

Observed values, with the application maximised:

```
scrim   1607x1125  override-redirect, _NET_WM_WINDOW_TYPE_DIALOG, WM_TRANSIENT_FOR = main
owner   1615x1129  WM-managed
dialog   845x546   WM-managed, viewable, healthy
```

---

## The fix

Make the scrim share its owner's stacking regime. One condition added to `is_window_managed()`,
plus a helper.

```diff
--- a/dlls/winex11.drv/window.c
+++ b/dlls/winex11.drv/window.c
@@
+/***********************************************************************
+ *              is_owned_scrim
+ *
+ * Check whether a popup covers its owner, i.e. whether it is a modal scrim
+ * rather than a menu, tooltip, drop-down or docked panel. set_style_hints()
+ * already advertises such a window as _NET_WM_WINDOW_TYPE_DIALOG with
+ * WM_TRANSIENT_FOR set, hints that only apply to managed windows.
+ *
+ * A window rejected here while still sized 0x0 is reconsidered later:
+ * X11DRV_WindowPosChanged() calls is_window_managed() again once the window
+ * has real rects, and window_set_managed() supports the unmanaged -> managed
+ * transition.
+ */
+static BOOL is_owned_scrim( HWND hwnd )
+{
+    HWND owner = NtUserGetWindowRelative( hwnd, GW_OWNER );
+    RECT win, own;
+    UINT dpi;
+
+    if (!owner) return FALSE;
+
+    dpi = NtUserGetDpiForWindow( owner );
+    if (!NtUserGetWindowRect( hwnd, &win, dpi )) return FALSE;
+    if (!NtUserGetWindowRect( owner, &own, dpi )) return FALSE;
+    /* a scrim covers a real window, not a 1x1 placeholder */
+    if (own.right - own.left < 100 || own.bottom - own.top < 100) return FALSE;
+
+    /* covers at least 90% of the owner in both axes */
+    return (win.right - win.left) * 10 >= (own.right - own.left) * 9 &&
+           (win.bottom - win.top) * 10 >= (own.bottom - own.top) * 9;
+}
+
 /***********************************************************************
  *		is_window_managed
  *
@@ static BOOL is_window_managed( HWND hwnd, UINT swp_flags, BOOL fullscreen )
         if (style & WS_SYSMENU) return TRUE;
         /* full-screen popup windows are managed */
         if (fullscreen) return TRUE;
+        /* a popup covering its owner is a modal scrim; it must share the owner's
+         * stacking regime, or the window manager can never raise the owner's own
+         * dialogs above it */
+        if (is_owned_scrim( hwnd )) return TRUE;
     }
     /* application windows are managed */
     ex_style = NtUserGetWindowLongW( hwnd, GWL_EXSTYLE );
```

### Why these criteria

The window must have an owner, the owner must be a real window (at least 100x100), and the
popup must cover at least 90% of that owner on both axes. Menus, tooltips, drop-downs and
docked panels are all far smaller than their owner and are therefore untouched.

### The `WS_EX_TOOLWINDOW` trap

A first version of this patch also excluded `WS_EX_TOOLWINDOW`, on the assumption that menus and
tooltips use that style and should be protected. **That silently breaks the fix.** The scrim is
itself a tool window — its extended style is `0x00080080`, i.e.
`WS_EX_TOOLWINDOW | WS_EX_LAYERED` — so the exclusion rejects the exact window being targeted,
and the patch compiles, installs and does nothing.

The exclusion is also unnecessary. Instrumenting the helper to log every popup it evaluates
shows the size criterion already doing the work, against a `1208x646` owner:

| window | size | owner | coverage | outcome |
|---|---|---|---|---|
| browser panel | 366x772 | 1208x646 | 30% of width | unmanaged |
| docked panel | 300x412 | 1208x646 | 25% of width | unmanaged |
| toolbar strip | 300x25 | 1208x646 | 25% of width | unmanaged |
| message tray | 1x1 | 1208x646 | — | unmanaged |
| **the scrim** | **1607x1125** | **1615x1129** | **99.5% x 99.6%** | **managed** |

Managing a tool window does **not** give it a title bar. `get_mwm_decorations_for_style()`
already returns no decorations for both `WS_EX_TOOLWINDOW` and `WS_EX_LAYERED`, so the only
thing that changes is that the window manager now orders it.

### Late promotion is supported

Some windows are evaluated before they have been sized, and a `0x0` window cannot satisfy the
coverage test. This is handled by Wine itself — `X11DRV_WindowPosChanged()` re-runs the check
once the window has real rects:

```c
if ((is_managed = is_window_managed( hwnd, swp_flags, fullscreen ))) make_owner_managed( hwnd );
if (!(data = get_win_data( hwnd ))) return;
if (is_managed) window_set_managed( data, TRUE );
```

and `window_set_managed()` supports the unmanaged → managed transition (it withdraws the window,
flips `override_redirect`, and re-maps it). The reverse transition is not supported, which is why
the patch only ever promotes.

---

## Building a patched Wine

The patch touches one file, but Wine must be built as a whole.

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
patchset, which is what makes the registry value

```
HKCU\Software\Wine   HideWineExports = Y
```

work. Without it Fusion detects that it is running under Wine and logs
`Fusion is running on GNU/Linux using WINE`. With it, Fusion's embedded Python reports
`os.name == 'nt'` as expected.

### Build

```bash
mkdir -p "$HOME/wine-build" && cd "$HOME/wine-build"

git clone --depth 1 --branch wine-11.19 https://github.com/wine-mirror/wine.git wine
git clone --depth 1 --branch v11.19 https://github.com/wine-staging/wine-staging.git

python3 wine-staging/staging/patchinstall.py --all --destdir="$PWD/wine"

# apply the patch above to wine/dlls/winex11.drv/window.c

mkdir build && cd build
../wine/configure --enable-archs=i386,x86_64 \
                  --prefix="$HOME/wine-build/install" \
                  --disable-tests
make -j"$(nproc)"
make install
```

A full build takes a few minutes on a modern multi-core machine. Because the patch touches a
single file, every subsequent iteration rebuilds in **seconds** — which matters a great deal
when debugging this, since the alternative is a full Fusion launch cycle per attempt.

### A note on glibc, if you build and run on different systems

Binaries linked against an older glibc run on a newer one, but not the reverse. If the build
system and the system running Fusion are not the same, build on whichever has the **older**
glibc. Verify with `ldd <install>/bin/wine | grep 'not found'` on the target before relying on it.

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

Do not debug this by starting Fusion and clicking. Reproduce the window arrangement with a
minimal Win32 program instead; the whole cycle then takes seconds.

`scrimtest.c` — creates an owner window, an oversized caption-less owned popup shown with
`SW_SHOWNOACTIVATE` (the scrim), and an owned dialog:

```c
#include <windows.h>

static LRESULT CALLBACK wndproc( HWND h, UINT m, WPARAM w, LPARAM l )
{
    if (m == WM_DESTROY) { PostQuitMessage(0); return 0; }
    if (m == WM_ERASEBKGND)
    {
        RECT r; HDC dc = (HDC)w;
        GetClientRect( h, &r );
        FillRect( dc, &r, (HBRUSH)GetWindowLongPtrW( h, GWLP_USERDATA ) );
        return 1;
    }
    return DefWindowProcW( h, m, w, l );
}

int WINAPI wWinMain( HINSTANCE inst, HINSTANCE prev, WCHAR *cmd, int show )
{
    WNDCLASSW wc = { 0 };
    MSG msg; HWND main_w, scrim, dlg; RECT mr;

    wc.lpfnWndProc = wndproc; wc.hInstance = inst;
    wc.hCursor = LoadCursorW( NULL, (const WCHAR *)IDC_ARROW );
    wc.lpszClassName = L"scrimtest";
    RegisterClassW( &wc );

    main_w = CreateWindowExW( 0, L"scrimtest", L"scrimtest-main",
                              WS_OVERLAPPEDWINDOW, 200, 150, 1200, 800,
                              NULL, NULL, inst, NULL );
    SetWindowLongPtrW( main_w, GWLP_USERDATA, (LONG_PTR)GetStockObject( GRAY_BRUSH ) );
    ShowWindow( main_w, SW_SHOW );
    UpdateWindow( main_w );
    GetWindowRect( main_w, &mr );

    /* the scrim: owned, no caption, covers the owner, never activated */
    scrim = CreateWindowExW( 0, L"scrimtest", L"scrimtest-scrim", WS_POPUP,
                             mr.left, mr.top - 30,
                             mr.right - mr.left, (mr.bottom - mr.top) + 30,
                             main_w, NULL, inst, NULL );
    SetWindowLongPtrW( scrim, GWLP_USERDATA, (LONG_PTR)GetStockObject( DKGRAY_BRUSH ) );
    ShowWindow( scrim, SW_SHOWNOACTIVATE );

    /* the dialog the scrim is supposed to sit behind */
    dlg = CreateWindowExW( WS_EX_DLGMODALFRAME, L"scrimtest", L"scrimtest-dialog",
                           WS_POPUP | WS_CAPTION | WS_SYSMENU,
                           mr.left + 400, mr.top + 250, 845, 546,
                           main_w, NULL, inst, NULL );
    SetWindowLongPtrW( dlg, GWLP_USERDATA, (LONG_PTR)GetStockObject( WHITE_BRUSH ) );
    ShowWindow( dlg, SW_SHOW );
    SetForegroundWindow( dlg );

    while (GetMessageW( &msg, NULL, 0, 0 ) > 0)
    {
        TranslateMessage( &msg );
        DispatchMessageW( &msg );
    }
    return 0;
}
```

```bash
x86_64-w64-mingw32-gcc -municode -mwindows -O1 -o scrimtest.exe scrimtest.c -lgdi32
```

`check_stack.py` — reports override-redirect state and true stacking order. `XQueryTree`
returns children bottom-to-top; managed windows are reparented into a frame, so look one level
down as well. Requires `xdotool` and `xwininfo`:

```python
#!/usr/bin/env python3
import ctypes, ctypes.util, subprocess, sys

x11 = ctypes.CDLL(ctypes.util.find_library("X11"))
x11.XOpenDisplay.restype = ctypes.c_void_p
x11.XOpenDisplay.argtypes = [ctypes.c_char_p]
x11.XDefaultRootWindow.restype = ctypes.c_ulong
x11.XDefaultRootWindow.argtypes = [ctypes.c_void_p]

def tree(dpy, w):
    root = ctypes.c_ulong(); parent = ctypes.c_ulong()
    kids = ctypes.POINTER(ctypes.c_ulong)(); n = ctypes.c_uint()
    if not x11.XQueryTree(ctypes.c_void_p(dpy), ctypes.c_ulong(w), ctypes.byref(root),
                          ctypes.byref(parent), ctypes.byref(kids), ctypes.byref(n)):
        return []
    out = [kids[i] for i in range(n.value)]
    x11.XFree(kids)
    return out

want = {}
for suffix in ("main", "scrim", "dialog"):
    for g in subprocess.run(["xdotool", "search", "--name", f"scrimtest-{suffix}$"],
                            capture_output=True, text=True).stdout.split():
        want[int(g)] = suffix
if not want:
    sys.exit("no scrimtest windows found")

dpy = x11.XOpenDisplay(None)
root = x11.XDefaultRootWindow(ctypes.c_void_p(dpy))

state, order = {}, []
for xid, role in want.items():
    info = subprocess.run(["xwininfo", "-id", str(xid)], capture_output=True, text=True).stdout
    state[role] = "Override Redirect State: yes" in info
    print(f"{role:<8} {xid:<12} override={'YES' if state[role] else 'no'}")

for top in tree(dpy, root):
    for cand in [top] + tree(dpy, top):
        if cand in want:
            info = subprocess.run(["xwininfo", "-id", str(cand)],
                                  capture_output=True, text=True).stdout
            if "Map State: IsViewable" in info:
                order.append(want[cand])

print("stacking, bottom -> top:", " < ".join(order))
print("scrim is", "override-redirect -> UNPATCHED" if state.get("scrim") else "WM-managed -> PATCHED")
if "scrim" in order and "dialog" in order:
    print("dialog is", "ABOVE the scrim -> FIXED" if order.index("dialog") > order.index("scrim")
                       else "BELOW the scrim -> BROKEN")
```

### Expected results

Run the reproducer under each build, swapping only `lib/wine/x86_64-unix/winex11.so`, so that
the comparison isolates a single file:

| build | scrim override-redirect | stacking, bottom → top | verdict |
|---|---|---|---|
| stock | **yes** | main < dialog < **scrim** | dialog buried — BROKEN |
| patched | no | main < **scrim** < dialog | dialog on top — FIXED |

**The stock result matters as much as the patched one.** If the unpatched build does not
reproduce the defect, the test is not exercising the right code path and a "fixed" result from
the patched build means nothing.

### Regression check on the real application

The risk in this patch is not the stacking change, it is the 90% heuristic promoting something
it should not. After installing, confirm that the application's genuine override-redirect
children are still unmanaged — the docked browser panel, toolbar strips and any 1x1 helper
windows should all still report `Override Redirect State: yes`, and the main window should still
appear in `_NET_CLIENT_LIST`:

```bash
xprop -root _NET_CLIENT_LIST
for w in $(xdotool search --class fusion360); do
  xwininfo -id "$w" | grep -q 'Map State: IsViewable' || continue
  echo "$w $(xdotool getwindowname $w)"
  xwininfo -id "$w" | grep 'Override Redirect State'
done
```

Also confirm 3D rendering survived: the launch log should contain `D3D11InternalCreateDevice`
entries and no `CreateSwapChainForComposition` errors.

---

## Emergency recovery, without the patch

If you hit the wedge on a stock Wine build, the scrim can be made click-transparent at runtime
by clearing its X input region, which lets clicks fall through to the dialog underneath. The
window stays visible. This is a stopgap, not a fix — modal input blocking is lost, and it must
be repeated for every new dialog:

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

Identify `WINDOW_ID` as the viewable, override-redirect child whose size is within ~90% of the
main window's. Leave the small override-redirect windows alone — those are legitimate panels.

Falling back to **File → New Design** (Ctrl+N) instead of the "+" button also avoids the
code path entirely, since that route does not create the scrim.

---

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

Not submitted upstream to WineHQ. The 90% coverage threshold is a heuristic, and upstream would
reasonably want to discuss whether promotion should instead be driven by the same condition that
`set_style_hints()` uses to decide the window type — which is the real inconsistency this patch
works around.

Contributions and a better criterion are welcome.
