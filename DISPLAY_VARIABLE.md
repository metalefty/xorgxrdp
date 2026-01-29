# Display Variable in xorg-server

## Overview

This document explains where the `display` variable is defined in xorg-server code and how xorgxrdp accesses it.

## Historical Context

The `display` variable was a **global variable** in older versions of xorg-server that contained the display name (e.g., ":0", ":1", etc.). In newer versions of xorg-server, this global was removed and replaced with a getter function from the DIX (Device Independent X) layer.

## Where is "display" defined?

### In Older xorg-server (before ~2014)
- **Type:** Global variable (`const char *display`)
- **Location:** In the xorg-server source code (defined in `dix/globals.c`, declared in `dix.h`)
- **Declaration:** `extern const char *display;` (implicitly available when including xorg-server headers)
- **Access:** Direct variable access: `display`
- **Scope:** Global, visible to all modules linked with xorg-server

### In Newer xorg-server (after ~2014)
- **Type:** Not a global variable anymore
- **Function:** `dixGetDisplayName()` from DIX layer
- **Location:** xorg-server dix layer
- **Access:** Via function call: `dixGetDisplayName(&pScreen)`

## How xorgxrdp Handles This

### Configuration Detection

The project's `configure.ac` detects which approach is available:

```m4
AC_CHECK_DECL([dixGetDisplayName],
    [AC_DEFINE([HAS_DIX_GET_DISPLAY_NAME], [1],
    [Define if dixGetDisplayName is available])],
    [],
    [#include <dix.h>])
```

### Implementation Pattern

Throughout xorgxrdp code, conditional compilation is used:

```c
#ifdef HAS_DIX_GET_DISPLAY_NAME
    const char *display = dixGetDisplayName(&dev->pScreen);
    if (display == NULL)
    {
        FatalError("rdpClientConInit: Can't get display from DIX layer");
    }
#else
    // For older xorg-server, 'display' is a global variable defined in
    // xorg-server's dix/globals.c and is implicitly available.
    // The variable is accessed directly without a local declaration.
    // It's in scope because xorg-server modules run in the server's address space.
#endif

// Common code that uses 'display' regardless of how it was obtained
errno = 0;
i = (int)strtol(display, &endptr, 10);
if (errno != 0 || display == endptr || *endptr != 0)
{
    FatalError("rdpClientConInit: can not run at non-integer display");
}
```

## Usage in xorgxrdp

### Primary Locations

1. **module/rdpClientCon.c** (lines 1816-1834)
   - Function: `rdpClientConInit()`
   - Purpose: Get display name for client connection initialization
   - Example usage:
     ```c
     #ifdef HAS_DIX_GET_DISPLAY_NAME
         const char *display = dixGetDisplayName(&dev->pScreen);
         if (display == NULL)
         {
             FatalError("rdpClientConInit: Can't get display from DIX layer");
         }
     #endif
     
     // Use display to create socket paths
     errno = 0;
     i = (int)strtol(display, &endptr, 10);
     if (errno != 0 || display == endptr || *endptr != 0)
     {
         FatalError("rdpClientConInit: can not run at non-integer display");
     }
     g_sprintf(dev->uds_data, "%s/xrdp_display_%s", socket_dir, display);
     ```

2. **module/rdpClientCon.c** (lines 1128-1134)
   - Function: `rdpStartAccelAssist()`
   - Purpose: Format display string for acceleration assistance
   - Example usage:
     ```c
     #ifdef HAS_DIX_GET_DISPLAY_NAME
         snprintf(text, 63, ":%s", dixGetDisplayName(&dev->pScreen));
     #else
         snprintf(text, 63, ":%s", display);
     #endif
     text[63] = 0;
     setenv("DISPLAY", text, 1);
     ```

### Related to rdpLoadLayout

While `rdpLoadLayout()` function in `xrdpkeyb/rdpKeyboard.c` doesn't directly use the `display` variable, it operates on structures that are part of the same xorg-server ecosystem:

- **DeviceIntPtr:** Device structure that contains keyboard device information
- **inputInfo.keyboard:** Global keyboard device from xorg-server's input subsystem
- **serverClient:** Global client structure representing the X server itself

These structures are all part of the xorg-server's device-independent layer (DIX) and share the same display context.

## Key Structures

### DeviceIntPtr (from inputstr.h in xorg-server)
```c
typedef struct _DeviceIntRec *DeviceIntPtr;
```
This is an opaque pointer to the internal device structure. It doesn't contain the display directly, but operates within the context of a display.

### InputInfo (from xorg-server)
```c
extern InputInfo inputInfo;
```
Global structure containing pointers to keyboard and pointer devices for the X server.

## Relationship to rdpLoadLayout

The `rdpLoadLayout()` function:
1. Takes an `rdpKeyboard *keyboard` parameter (which contains `DeviceIntPtr device`)
2. Calls `reload_xkb(keyboard->device, &set)`
3. Calls `reload_xkb(inputInfo.keyboard, &set)`

Both device pointers operate in the context of the X server's display, but the function doesn't need direct access to the `display` variable because:
- The XKB (X Keyboard Extension) functions work with device structures
- The display context is implicit through the server's global state
- The keyboard devices already know which display they belong to

## Summary

The `display` variable represents the X11 display name (e.g., "0" for ":0", "1" for ":1"):

- **Old xorg-server (pre-2014):** `display` was a global `const char*` variable defined in xorg-server's `dix/globals.c`
- **New xorg-server (post-2014):** Use `dixGetDisplayName(&pScreen)` function from the DIX layer to get the display name
- **xorgxrdp:** Uses conditional compilation (`HAS_DIX_GET_DISPLAY_NAME`) to support both approaches
- **rdpLoadLayout:** Doesn't directly use `display`, but operates on devices that exist within the display's context

## Additional Context

### Why the Change Happened

The xorg-server developers removed the global `display` variable to:
1. Improve encapsulation and reduce global state
2. Support multi-display scenarios more cleanly
3. Make the codebase more maintainable

### Where to Find xorg-server Source

The `display` variable (in older versions) and `dixGetDisplayName()` function (in newer versions) are part of the xorg-server source code:

- **Repository:** https://gitlab.freedesktop.org/xorg/xserver
- **Old global definition:** `dix/globals.c` (removed in later versions)
- **New getter function:** `dix/dixfonts.c` or similar (implementation of `dixGetDisplayName()`)
- **Header:** `include/dix.h` contains declarations

### Technical Details

**DeviceIntPtr and the Display Context:**

- `DeviceIntPtr` (Device Internal Pointer) is defined in `inputstr.h` in xorg-server
- It points to `struct _DeviceIntRec`, which contains device state
- The device doesn't store the display name directly
- Instead, the device is associated with a `ScreenPtr` (via `pScreen`)
- The screen knows which display it belongs to
- `dixGetDisplayName(&pScreen)` retrieves the display name through this relationship

**Why rdpLoadLayout doesn't need display:**

The `rdpLoadLayout()` function operates at the keyboard device level through XKB (X Keyboard Extension) APIs. These APIs work with device structures (`DeviceIntPtr`) that already know their display context through the screen association. The function doesn't need to explicitly reference the display name because:

1. It calls `reload_xkb()` with device pointers
2. The XKB subsystem uses the device's internal screen/display association
3. The display context is implicit in all X server operations
