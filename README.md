# Pintinit

### Because normal init systems are too normal.

Most init systems follow conventions. They're predictable, structured, and generally behave exactly as you'd expect.

**Pintinit has decided to explore alternative possibilities.**

Pintinit is an experimental Linux init system written in Python and compiled using Nuitka. It starts services, watches processes, and accepts commands through a Unix domain socket. It also has a few ideas about what a boot message should look like.

Its philosophy is simple:

> Just because something is unconventional doesn't mean we shouldn't try it.

Does it boot? It has been tested in a QEMU-based Linux environment.

Is it finished? Not even close.

Should you replace your existing init system with it? That's a decision between you, your computer, and your recovery environment.

**Welcome to Pintinit.**

*The system would like permission to get permission to start.*

## What is Pintinit?

Pintinit is an early-stage Linux init system intended to run as **PID 1**. It launches configured services, checks whether they are still running, and attempts to restart processes that exit unexpectedly.

The main implementation is Python, packaged as a standalone distribution using **Nuitka**. A separate command-line client communicates with the init process over a local Unix datagram socket.

This is a real experiment with real processes, not a simulation of an init system. It is also **not production-ready**. Use a virtual machine or another disposable environment for testing.

## Current features

- **PID 1 operation:** the main programme checks that it is running as PID 1.
- **Configurable services:** startup commands are read from `/etc/pintinit/config.py`.
- **Process monitoring:** services are polled and unexpected exits trigger restart attempts.
- **One-shot service option:** service definitions can indicate that a successful exit is expected. The monitoring loop does not restart a one-shot service that exits with status 0.
- **Local control socket:** Pintinit listens at `/run/pintinit.sock` for JSON-encoded command arguments.
- **Shutdown command:** the control interface can request a service shutdown sequence followed by a Linux power-off syscall.
- **Boot and runtime output:** startup messages appear on the console, with additional logging support for `/dev/tty8` and `/var/log/pintinit.log`.
- **QEMU test environment:** the repository contains scripts for building a small initramfs and booting it in a virtual machine.

Some of these features are incomplete. In particular, the command interface currently implements `shutdown`, not a full service-management command set.

## Requirements

The supplied scripts target a Linux development environment, particularly **x86-64** for the QEMU test setup.

You will need:

- Python 3 with `venv` and `pip`
- GCC and the dependencies required by Nuitka
- BusyBox
- QEMU (`qemu-system-x86_64`)
- A Linux kernel at `/boot/vmlinuz-linux` for the current test script
- `patchelf`, which Nuitka requires for standalone builds on Linux
- Standard build utilities including Bash, `cpio`, `gzip`, `readelf`, `ldconfig`, `agetty`, and `login`

The build script installs Python requirements from `requirements.txt` into a local virtual environment. Other system dependencies must be available separately.

**Note:** The current test script assumes a particular host layout, kernel location and library architecture. It is not yet a portable one-command installer.

## Building Pintinit

Clone the repository:

```sh
git clone https://github.com/Enderman-brewer/pintinit.git
cd pintinit
```

Build the standalone distributions:

```sh
./build.sh
```

The script creates a Python virtual environment, installs the requirements, compiles the main init programme and control utility using Nuitka, and assembles `final_dist/`.

The main init executable is placed under:

```text
final_dist/opt/pintinit/main.bin
```

The generated distribution also includes a shell-based `/sbin/init` launcher and a sample configuration file.

### Testing in QEMU

Run:

```sh
./test.sh
```

This builds a test initramfs from the compiled distribution and launches QEMU. It removes and recreates the local `fakefs/` directory while preparing the image.

To check prerequisites, build, and launch the test in sequence:

```sh
./all.sh
```

**Warning:** `all.sh` runs the VM test as its final step. It is not merely a compilation command.

The test image contains a minimal BusyBox environment, login utilities, the compiled Pintinit programme, and example service definitions. It is designed for experimentation rather than installation on an existing machine.

## Configuring services

Pintinit loads its service configuration from:

```text
/etc/pintinit/config.py
```

The repository includes `config-skel.py`, which the build scripts copy into that location.

A service entry currently follows this format:

```python
services = [
    [
        "basic-login-test@tty1",
        "Launch a basic shell",
        ["/sbin/agetty", "-o", "-p -- \\u", "--noclear", "tty1", "linux"],
        False,
        [],
    ],
]
```

| Position | Meaning | Current behaviour |
|---|---|---|
| `0` | Service name | Used to identify the service |
| `1` | Description | Present in the configuration, not yet used by the service manager |
| `2` | Command | Executable and arguments passed to `subprocess.Popen()` |
| `3` | One-shot flag | Passed to the start routine and used by the monitoring loop, so a status-0 exit is not treated as a crash |
| `4` | Requirements | Reserved for dependencies; not yet enforced |

Commands must be provided as a list of separate arguments. Pintinit does not automatically run them through a shell.

The current example configuration starts login prompts on `tty1` and `ttyS0`. The sample image is for testing; do not treat its account or login configuration as suitable for a secure installation.

## Controlling Pintinit

The source file `mainctl.py` provides the control utility. It sends the command-line arguments as a JSON list to:

```text
/run/pintinit.sock
```

For example, the implemented command is:

```sh
pintinitctl shutdown
```

**Naming note:** The Nuitka build of `mainctl.py` produces the binary `pintinitctl`, which `build.sh` copies to `final_dist/sbin/pintinitctl`.

The socket is created with permissions `0600`, so control normally requires the appropriate privileges.

The current protocol is **one-way**: the client sends a command but does not receive confirmation that the operation completed. The client forwards any arguments it is given, but the command handler only acts on `shutdown`. `start`, `stop`, `restart`, and `status` are not implemented.

## How it works

At a high level:

1. A small shell launcher mounts essential virtual filesystems and executes the compiled Pintinit programme.
2. Pintinit verifies that it is running as PID 1.
3. It imports the Python configuration and starts the listed services.
4. It creates a non-blocking Unix datagram socket for control commands.
5. Its main loop checks service processes, handles pending commands, and sleeps briefly before repeating.

The current polling interval is **10 milliseconds**. Some operations, including service startup and shutdown, block the loop for longer.

Pintinit currently uses a straightforward single-process control loop rather than a more elaborate event-driven supervisor. Straightforward does not necessarily mean finished.

## Known limitations

Pintinit is under active development. Current limitations include:

- **Service restart tracking:** the failed-service data structure is inconsistent across some code paths, which can cause errors during repeated failures. The fix is in PR #1 and is not yet merged into this branch.
- **Restart policy:** there is no reliable restart backoff or complete crash-limit enforcement.
- **Dependencies:** service requirement lists are defined but not processed.
- **Command coverage:** only `shutdown` is handled; other management commands remain to be implemented.
- **Command responses:** the client cannot currently verify that Pintinit acted on a request.
- **Blocking operations:** service start and stop routines can delay command handling.
- **Portability:** the test scripts make x86-64 and host-specific assumptions, including the kernel path.
- **Installation:** there is no supported procedure for safely replacing an existing system init.

There may be additional bugs. Finding them is part of the experiment. Discovering them on your primary machine is not the recommended method.

## Project layout

```text
pintinit/
├── main.py          # PID 1 implementation and service supervision
├── mainctl.py       # Unix socket command-line client
├── config-skel.py   # Example service configuration
├── requirements.txt # Python build dependencies
├── reqs.sh          # Basic prerequisite checks
├── build.sh         # Nuitka compilation and distribution assembly
├── test.sh          # Initramfs creation and QEMU test
└── all.sh           # Prerequisites, build, then test
```

`build_ctl/` and `final_dist/` are generated, but they are committed to the repository. `build/`, `fakefs/`, and `venv/` are git-ignored and created locally.

## Roadmap

These are possible development directions, **not promises of existing functionality**:

- Complete and standardise the `pintinitctl` command interface.
- Implement start, stop, restart and status commands.
- Repair one-shot service handling and failure tracking.
- Add robust restart limits and backoff.
- Honour service dependencies and define startup ordering.
- Improve logging, command acknowledgements and error reporting.
- Make build and test workflows less dependent on the host distribution.
- Document a safer installation process once the project is ready for one.

## Contributing

Bug reports, tests, documentation improvements and focused pull requests are welcome through the GitHub repository.

When reporting an issue, include the steps to reproduce it, relevant console output, and whether you were running inside the supplied QEMU environment or another Linux setup. For boot problems, kernel and architecture details are especially useful.

Please keep contributions technically useful, even when the software itself is being unusually creative.

## Why Pintinit?

Because experimenting with the first userspace process is an interesting way to learn how Linux starts, supervises processes, and shuts down.

Because conventions are useful, but understanding *why* they exist is more useful than following them blindly.

And because an init system can be a serious engineering experiment without having a serious personality.

**Pintinit: taking responsibility for PID 1, with varying degrees of confidence.**
