# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

This is the [Asterisk](https://www.asterisk.org) open source PBX and telephony toolkit — a large C codebase that acts as middleware between telephony/internet channels (bottom layer) and telephony/internet applications (top layer).

## Build Commands

```bash
# First-time setup
./contrib/scripts/install_prereq install   # Install system dependencies
./configure                                 # Detect system capabilities
make menuselect                             # Select modules to build (optional)

# Build
make                                        # Build all selected modules
make -j$(nproc)                            # Parallel build (faster)

# Install
make install                               # Install binaries
make samples                               # Install sample config files (overwrites existing!)
make basic-pbx                             # Install minimal working config

# Developer build (enables extra compiler checks and runtime assertions)
AST_DEVMODE=yes make

# Build with strict warnings
AST_DEVMODE=yes AST_DEVMODE_STRICT=yes make

# Generate Doxygen API docs
make progdocs

# Clean
make clean                                 # Clean build artifacts
make distclean                             # Clean including configure output
```

## Running Asterisk

```bash
# Foreground with verbose console (development)
asterisk -vvvc

# Connect to running instance's CLI
asterisk -r

# Execute a single CLI command
asterisk -rx "core show settings"

# Remote execute against custom config
asterisk -rx "core show version" -C /path/to/asterisk.conf
```

## Running Tests

Unit tests are compiled into Asterisk itself (in `tests/`) and run via the CLI:

```bash
# From a running Asterisk instance:
*CLI> test execute all
*CLI> test execute category /category/name/
*CLI> test show results failed
*CLI> test generate results xml /path/to/output.xml
```

The CI runner (`tests/CI/runUnittests.sh`) starts Asterisk, runs all unit tests, saves XML results, then stops Asterisk. To run a specific test category programmatically:

```bash
UNITTEST_COMMAND="test execute category /main/astobj2/" tests/CI/runUnittests.sh
```

## Architecture

### Source Tree Layout

Asterisk's build system compiles these module subdirectories into loadable `.so` files:

| Directory | Contents |
|-----------|----------|
| `main/` | Core Asterisk daemon, foundational APIs (astobj2, bridge, channel, config, etc.) |
| `channels/` | Channel drivers: `chan_pjsip.c`, `chan_iax2.c`, `chan_dahdi.c`, etc. |
| `pbx/` | Dialplan engines: `pbx_config.c` (standard .conf), `pbx_ael.c` (AEL), `pbx_lua.c` |
| `apps/` | Dialplan applications (`app_dial.c`, `app_confbridge.c`, `app_voicemail.c`, etc.) |
| `res/` | Resource modules, including the critical `res_pjsip/` subsystem |
| `funcs/` | Dialplan functions (`FILTER()`, `CALLERID()`, `DB()`, etc.) |
| `bridges/` | Bridge technologies for connecting channels |
| `codecs/` | Audio codec implementations |
| `formats/` | Audio file format handlers |
| `cdr/` | Call Detail Record backends |
| `cel/` | Channel Event Logging backends |
| `addons/` | Additional modules with different licensing |
| `tests/` | In-process unit tests (compiled with `AST_TEST_DEFINE` macro) |
| `include/asterisk/` | Public API headers |
| `third-party/` | Bundled pjproject (SIP stack), jansson (JSON), libjwt |

### Key Subsystems

**Channel Layer** (`main/channel.c`, `include/asterisk/channel.h`): The `ast_channel` struct is the central abstraction for a call leg. Everything connects through channels.

**Bridge Layer** (`main/bridge.c`, `bridges/`): Connects two or more channels together. Bridge technologies in `bridges/` provide different mixing strategies (simple passthrough, softmix for conferencing, native RTP for direct media).

**PJSIP Stack** (`res/res_pjsip/`, `channels/pjsip/`, `res/res_pjsip_*.c`): The primary SIP implementation. `res_pjsip.so` is the core; supplementary functionality (registration, subscriptions, OPTIONS, etc.) lives in separate `res_pjsip_*.c` files. Configuration uses Sorcery (object abstraction layer).

**Dialplan** (`pbx/pbx_config.c`, `main/pbx.c`): Routes calls through contexts/extensions/priorities. Applications (`apps/`) implement dialplan commands. Functions (`funcs/`) provide read/write variable access.

**AstObj2** (`main/astobj2.c`): Reference-counted object system used pervasively. Containers (hash tables, red-black trees) are built on it.

**Sorcery** (`main/sorcery.c`): Object persistence abstraction — config files, realtime databases, and in-memory are all backends. Used heavily by PJSIP for configuration.

**ARI** (`res/res_ari.c`, `res/ari/`): Asterisk REST Interface — HTTP+WebSocket API for external call control. Generated from `rest-api/` JSON definitions via `rest-api-templates/`.

**AGI** (`res/res_agi.c`, `agi/`): Asterisk Gateway Interface — stdio-based external call control for scripts.

**AMI** (`main/manager.c`): Asterisk Manager Interface — TCP text protocol for management and event monitoring.

### Module System

Modules are dynamically loaded `.so` files. Each exports `ast_module_info` with load/unload/reload callbacks. The `menuselect` tool controls which modules are compiled. Module dependencies are declared via `AST_MODULE_SUPPORT_*` and `AST_MODULE_DEPENDS_*` macros.

### Config File Format

All Asterisk `.conf` files share a common INI-like format: sections in `[]`, key-value with `=` or `=>`, comments with `;`. The `include/asterisk/config.h` API parses these.

## Coding Conventions

- C99 with GCC extensions; minimum GCC 4.1
- Doxygen comments on all public API functions (`/*! \brief ... */`)
- `AST_TEST_DEFINE(test_name)` macro for unit tests; register with `AST_TEST_REGISTER()`
- Thread safety via `ast_mutex_t`, `ast_rwlock_t`, or AstObj2 locking
- Memory: use `ast_malloc`/`ast_free`/`ast_strdup` (tracked by `astmm` in dev mode)
- Coding guidelines: https://docs.asterisk.org/Development/Policies-and-Procedures/Coding-Guidelines/

## Issue Tracking

Bugs: https://github.com/asterisk/asterisk/issues
