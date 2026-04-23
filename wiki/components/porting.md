---
type: component
parent_module: "[[components/base|base]]"
path: "src/base/"
status: developing
purpose: "OS portability layer: POSIX↔Win32 shims, portable atomics, timeval, numeric constants, and cross-platform process/library loading utilities"
key_files:
  - "porting.h (master platform abstraction: #define shims, type aliases, constants)"
  - "porting.c (runtime implementations: poll() on Windows, atomic ops, etc.)"
  - "porting_inline.hpp (inline C++ portability helpers)"
  - "process_util.c/h (cross-platform process creation/termination)"
  - "dynamic_load.c/h (dlopen/dlsym on Linux, shl_load on HP-UX, LoadLibrary on Windows)"
  - "cubrid_getopt_long.c/h (CUBRID's own getopt_long implementation)"
tags:
  - component
  - cubrid
  - porting
  - portability
related:
  - "[[components/base|base]]"
  - "[[components/error-manager|error-manager]]"
  - "[[components/system-parameter|system-parameter]]"
  - "[[Build Modes (SERVER SA CS)]]"
created: 2026-04-23
updated: 2026-04-23
---

# `porting` — Platform Abstraction Layer

`porting.h` is the gateway header for any platform-specific concern in CUBRID. It provides `#define` shims that map POSIX names to their Win32 equivalents, portable numeric constants, GCC version detection, DLL export/import macros, and type definitions for platform-specific types.

> [!key-insight] Included transitively via system_parameter.h
> `system_parameter.h` includes `porting.h`, and `system_parameter.h` is included by nearly every engine file. This means `porting.h`'s macros and constants are effectively always in scope without explicit inclusion.

## Win32 Function Shims

When `WINDOWS` is defined, `porting.h` maps POSIX names to Win32 equivalents:

```c
#define sleep(sec)            Sleep(1000*(sec))
#define usleep(usec)          Sleep((usec)/1000)
#define mkdir(dir, mode)      _mkdir(dir)
#define getpid()              _getpid()
#define snprintf              _sprintf_p
#define strcasecmp(s1,s2)     _stricmp(s1, s2)
#define strncasecmp(s,t,n)    _strnicmp(s, t, n)
#define lseek(fd,off,org)     _lseeki64(fd, off, org)
#define fseek(fd,off,org)     _fseeki64(fd, off, org)
#define ftruncate(fd,sz)      _chsize_s(fd, sz)
#define strdup(src)           _strdup(src)
#define popen                 _popen
#define pclose                _pclose
#define strtok_r              strtok_s
#define strtoll               _strtoi64
#define stat                  _stati64
#define fstat                 _fstati64
#define ftell                 _ftelli64
#define vsnprintf             cub_vsnprintf
#define printf                _printf_p
#define fprintf               _fprintf_p
```

Windows also requires a custom `poll()` implementation (provided in `porting.c`) and custom `mktime` for 32-bit.

## Numeric Constants

```c
#define ONE_K     1024
#define ONE_M     1048576
#define ONE_G     1073741824
#define ONE_T     1099511627776LL
#define ONE_P     1125899906842624LL

#define ONE_SEC   1000      // milliseconds
#define ONE_MIN   60000
#define ONE_HOUR  3600000

#define CTIME_MAX 64        // safe buffer size for ctime_r output
```

These constants are used throughout the engine for size calculations, timeout values, and buffer sizing.

## DLL Export/Import (`EXPORT_IMPORT`)

```c
#if defined (WINDOWS)
  #ifdef CUBRID_EXPORTING
    #define EXPORT_IMPORT __declspec(dllexport)
  #else
    #define EXPORT_IMPORT __declspec(dllimport)
  #endif
#else
  // all symbols are exported by default on Linux/macOS
  #define EXPORT_IMPORT
#endif
```

Used in public API declarations visible across DLL boundaries (broker ↔ CAS, CCI client headers).

## GCC Version Detection

```c
#if defined (__GNUC__) && defined (__GNUC_MINOR__) && defined (__GNUC_PATCHLEVEL__)
#define CUB_GCC_VERSION (__GNUC__ * 10000 + __GNUC_MINOR__ * 100 + __GNUC_PATCHLEVEL__)
#endif
```

Used for version-conditional compiler intrinsics and attributes. On non-GCC compilers:
```c
#if !defined (__GNUC__)
#define __attribute__(X)   // no-op stub
#endif
```

## File Descriptor Flags (`O_*` remapping)

On Windows, file open flags are remapped to always include `_O_BINARY`:
```c
#define O_CREAT   _O_CREAT|_O_BINARY
#define O_RDWR    _O_RDWR|_O_BINARY
#define O_RDONLY  _O_RDONLY|_O_BINARY
#define O_TRUNC   _O_TRUNC|_O_BINARY
#define O_EXCL    _O_EXCL|_O_BINARY
```

Windows does not differentiate text vs binary mode by default — CUBRID always uses binary mode for database files.

## `dynamic_load.c/h` — Shared Library Loading

```c
// Platform-independent handles
void *dl_open(const char *libname);         // dlopen / shl_load / LoadLibrary
void *dl_sym(void *handle, const char *sym); // dlsym / shl_findsym / GetProcAddress
int   dl_close(void *handle);               // dlclose / shl_unload / FreeLibrary
const char *dl_error(void);                 // dlerror / equivalent
```

Used for runtime loading of optional components (e.g., Java PL bridge, optional locale libraries).

## `process_util.c/h` — Cross-Platform Process Management

```c
// Start a child process
int putil_exec(const char *path, const char *argv[], ...);

// Terminate process by PID
int putil_kill(int pid, int signal);

// Check if process is alive
bool putil_is_running(int pid);
```

Used by the broker to manage CAS worker processes.

## `cubrid_getopt_long`

CUBRID ships its own `getopt_long` implementation for portability across platforms where the system `getopt_long` may not exist (older POSIX, Windows). Used by all command-line utilities (`cubrid`, `csql`, `cub_admin`).

## Memory Size Validation

```c
#define MEM_SIZE_IS_VALID(size) \
  (((long long unsigned)(size) <= ULONG_MAX) \
   || (sizeof(long long unsigned) <= sizeof(size_t)))
```

Guards against silent truncation when converting between `size_t` and smaller types on 32-bit platforms.

## POSIX Poll Emulation (Windows)

`porting.c` provides a `poll()` implementation for Windows using `WSAPoll` (Vista+) or a `select()`-based fallback. The `struct pollfd` definition is also provided for older Windows SDK targets:

```c
struct pollfd {
  SOCKET fd;
  SHORT events;
  SHORT revents;
};
int poll(struct pollfd *fds, nfds_t nfds, int timeout);
```

## Related

- [[components/base|base]] — parent hub
- [[components/system-parameter|system-parameter]] — includes `porting.h`; provides platform-aware config reading
- [[Build Modes (SERVER SA CS)]] — `SERVER_MODE`/`SA_MODE`/`CS_MODE` guards interact with porting shims
- Source: [[sources/cubrid-src-base|cubrid-src-base]]
