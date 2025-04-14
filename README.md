## vscode-mem - solve Remote-SSH disconnection issues in Visual Studio Code

Are you experiencing mysterious Remote-SSH disconnections in Visual Studio Code?
This issue is often caused by Node.js reserving excessive virtual memory,
exceeding system limits in shared environments. This repository provides a
script to fix the problem by adjusting memory settings, ensuring stable
Remote-SSH connections.

Visual Studio Code may encounter issues in shared environments due to a design
fault in Node.js. Node.js reserves a large virtual memory area, even though the
actual memory usage of Visual Studio Code is typically only a few hundred
megabytes or, at worst, a few gigabytes. This reservation can cause problems if
the virtual memory limit is set below 64 GB, which is common in shared
environments.

Virtual memory is a memory management feature in modern operating systems that
provides each process with its own private address space, often much larger than
the available physical memory (RAM). This space is virtual, meaning it is mapped
to physical memory or disk only when actually used. While Linux typically allows
large virtual memory reservations, most of this memory remains unallocated
unless accessed. However, on shared systems like HPC or login nodes,
administrators often configure strict memory usage limits to protect resources.
If a process tries to reserve more virtual memory than allowed, it crashes
immediately, even if the actual memory usage is low.

Node.js, which powers the backend of Visual Studio Code, aggressively reserves
virtual memory at startup for purposes such as JIT compilation, large typed
arrays, and V8 heap segments. This behavior is especially common in
Electron-based environments like VS Code, where extensions like GitHub Copilot
are dynamically loaded. Even though Node.js may only use a few hundred megabytes
of memory, it can reserve over 64 GB of virtual memory, which exceeds the limits
set on many shared systems.

When the virtual memory limit is insufficient, the remote connection may restart
multiple times within a few minutes and eventually fail. The Visual Studio Code
server logs may show the following error:

```
[00:29:29.799] [server] # Fatal process OOM in Failed to reserve virtual memory for CodeRange
```

This is a known issue, and there are no plans to fix it, as noted in the related
GitHub issue:
[microsoft/vscode#115236](https://github.com/microsoft/vscode/issues/115236).

This script provides a way to inspect and adjust system resource limits to
prevent such issues. By analyzing the output of `ulimit -a`, you can identify
bottlenecks and reconfigure your system to ensure stable operation.

```bash
ulimit -a
```

```text
core file size          (blocks, -c) unlimited
data seg size           (kbytes, -d) unlimited
scheduling priority             (-e) 0
file size               (blocks, -f) unlimited
pending signals                 (-i) 4124130
max locked memory       (kbytes, -l) unlimited
max memory size         (kbytes, -m) 67108864
open files                      (-n) 131072
pipe size            (512 bytes, -p) 8
POSIX message queues     (bytes, -q) 819200
real-time priority              (-r) 0
stack size              (kbytes, -s) 8192
cpu time               (seconds, -t) unlimited
max user processes              (-u) 4096
virtual memory          (kbytes, -v) 67108864   # <- This is causing the problem
file locks                      (-x) unlimited
```

## How This Script Works

This script attempts to address the virtual memory issue by modifying the
startup files of the VS Code server. Specifically, it adds the
`--max-old-space-size` argument to the Node.js process used by the VS Code
server. This argument restricts the amount of memory reserved by Node.js,
ensuring that it stays within the limits of the system's virtual memory
configuration.

### Command-Line Options

This script supports various command-line options to customize its behavior. For
a full list of available options and their usage, run:

```bash
./vscode-mem --help
```

This will display detailed information about the supported arguments, including
how to set the `--max-old-space-size`, specify a custom Node.js application
name, or define a custom VS Code Server directory.

### Memory Usage Information

In addition to modifying the startup files, the script provides detailed memory
usage information for the Node.js processes associated with the VS Code server.
For example:

```text
Inspecting memory usage for Node.js processes:
PID 63453:  VmPeak 11648 MB, VmSize 11557 MB, VmHWM 185 MB, VmRSS  94 MB
PID 65122:  VmPeak 11504 MB, VmSize 11440 MB, VmHWM  62 MB, VmRSS  62 MB
PID 108673: VmPeak 32083 MB, VmSize 32058 MB, VmHWM 139 MB, VmRSS 120 MB
PID 108902: VmPeak 11516 MB, VmSize 11516 MB, VmHWM  45 MB, VmRSS  40 MB
PID 171466: VmPeak  1038 MB, VmSize   972 MB, VmHWM  39 MB, VmRSS  38 MB

Maximum virtual memory size: 65536 MB

Memory term explanations:
- VmPeak: Peak virtual memory size used by the process.
- VmSize: Current virtual memory size allocated to the process.
- VmHWM: Peak resident set size ("high water mark"), the maximum physical memory used.
- VmRSS: Current resident set size, the physical memory currently used.

To restrict memory usage, run:
  vscode-mem --set-max-old-space-size=N

Replace N with the desired memory limit in MB (default: 4096).
```

Here is what the memory terms mean:
- **VmPeak**: The peak virtual memory size used by the process.
- **VmSize**: The current virtual memory size allocated to the process.
- **VmHWM**: The peak resident set size ("high water mark"), which is the maximum physical memory used.
- **VmRSS**: The current resident set size, representing the physical memory currently in use.

### Example Usage

To restrict the memory usage of the VS Code server, you can run the script with
the `--set-max-old-space-size` option:

```bash
./vscode-mem --set-max-old-space-size=8192
```

This will modify the VS Code server's startup script to include the
`--max-old-space-size=8192` argument, limiting Node.js to use a maximum of 8 GB
of memory.

By using this script, you can ensure that the VS Code server operates reliably
in environments with strict virtual memory limits.

### Additional Considerations for GitHub Copilot

While this script addresses memory issues with the VS Code server, you may
encounter additional problems when using GitHub Copilot in remote environments.
These issues arise because Copilot extensions are designed to run in the UI
environment, and running them remotely can lead to instability.

To overcome these problems, you can configure GitHub Copilot to run from the
host by adding the following settings to
`$HOME/.vscode-server/data/Machine/settings.json`:

```json
"remote.extensionKind": {
  "GitHub.copilot": [
    "ui"
  ],
  "GitHub.copilot-chat": [
    "ui"
  ]
}
```

This ensures that the Copilot extensions run in the local UI environment,
avoiding potential issues in the remote environment.

## Installation Instructions

To install and use this script, follow these steps:

1. Clone the repository to a local directory:
   ```bash
   git clone https://github.com/your-repo/vscode-mem.git
   cd vscode-mem
   ```

2. Create a symbolic link to the script in a directory included in your `PATH`
   (e.g., `/usr/local/bin`):
   ```bash
   ln -s "$(pwd)/vscode-mem" /usr/local/bin/vscode-mem
   ```

3. (Optional) Add the script to your `.bash_profile` or `.bashrc` to
   automatically patch the VS Code server files whenever they are updated:
   ```bash
   echo 'vscode-mem --set-max-old-space-size=8192' >> ~/.bash_profile
   ```

   Replace `8192` with your desired memory limit in MB.

4. Reload your shell configuration to apply the changes:
   ```bash
   source ~/.bash_profile
   ```

Now, the script is ready to use. You can run it manually or let it automatically
patch the VS Code server files during startup.

## License

This project is licensed under the MIT License. See the full license text below:

```
MIT License

Copyright (c) 2025 Jukka Aho

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```
