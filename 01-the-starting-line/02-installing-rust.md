# Installing Rust 🔧

> **Let's get Rust on your machine. This takes about 5 minutes, and we'll walk through every single step.**

---

## Table of Contents

- [What We're Installing](#what-were-installing)
- [Installing on Windows](#installing-on-windows)
- [Installing on macOS](#installing-on-macos)
- [Installing on Linux](#installing-on-linux)
- [Verifying Your Installation](#verifying-your-installation)
- [Understanding What Got Installed](#understanding-what-got-installed)
- [Updating Rust](#updating-rust)
- [Uninstalling Rust](#uninstalling-rust)
- [Setting Up Your Editor](#setting-up-your-editor)
- [Troubleshooting](#troubleshooting)
- [Summary](#summary)

---

## What We're Installing

When you install Rust, you actually get **three things**:

| Tool | What It Does |
|------|-------------|
| **rustc** | The Rust compiler — turns your code into an executable |
| **cargo** | The package manager and build system (think: npm, pip, maven — but better) |
| **rustup** | The toolchain manager — installs and updates Rust itself |

We install them all at once using a tool called **rustup** (yes, same name as this tutorial series!).

### What's a "Toolchain"?

A toolchain is a set of tools needed to compile and run code. The Rust toolchain includes:

```
rustup (manages everything)
  └── stable toolchain (default)
        ├── rustc (compiler)
        ├── cargo (package manager)
        ├── rustfmt (code formatter)
        ├── clippy (linter / code analyzer)
        ├── rust-std (standard library)
        └── rust-docs (offline documentation)
```

Don't worry about all of these now — just know that one install gets you everything.

### What Is a Compiler, Really?

If you're new to compiled languages, here's what a compiler does: it **translates** human-readable code into machine code (binary instructions) that your CPU can execute directly.

Think of it like a translator at the United Nations:
- You write your speech in Rust (human language)
- The compiler translates it to x86-64 or ARM assembly (CPU language)
- The CPU reads and executes the translated version

The key difference from an interpreter (Python, JavaScript):
```
Compiler (Rust, C, Go):
  1. Read ALL your code
  2. Translate it ALL to machine code
  3. Produce an executable file
  4. You run the executable — the compiler is not involved anymore
  
  Analogy: A translator translates a whole book, then you read the translated book.

Interpreter (Python, Ruby):
  1. Read one line of your code
  2. Execute it immediately
  3. Read the next line
  4. Execute it... (repeat)
  
  Analogy: A translator sits next to you and translates each sentence as you read.
```

The compiled approach is faster because the translation happens once (at compile time), not every time you run the program. It also allows the compiler to see your entire program and optimize globally — an interpreter can only see one line at a time.

---

## Installing on Windows

### Step 1: Download the Installer

Go to [https://rustup.rs](https://rustup.rs) and download `rustup-init.exe`.

> **Alternative:** If you have `winget` (Windows Package Manager), you can run:
> ```powershell
> winget install Rustlang.Rustup
> ```

### Step 2: Run the Installer

1. Double-click `rustup-init.exe`
2. You'll see a terminal window with options. It looks like this:

```
Welcome to Rust!

This will download and install the official compiler for the Rust
programming language, and its package manager, Cargo.

Rustup metadata and toolchains will be installed into the Rustup
home directory, located at:

  C:\Users\YourName\.rustup

This can be modified with the RUSTUP_HOME environment variable.

The Cargo home directory is located at:

  C:\Users\YourName\.cargo

This can be modified with the CARGO_HOME environment variable.

The cargo, rustc, rustup and other commands will be added to
Cargo's bin directory, located at:

  C:\Users\YourName\.cargo\bin

This path will then be added to your PATH environment variable by
modifying the HKEY_CURRENT_USER/Environment/PATH registry key.

Current installation options:

   default host triple: x86_64-pc-windows-msvc
     default toolchain: stable
               profile: default
  modify PATH variable: yes

1) Proceed with standard installation (default - just press enter)
2) Customize installation
3) Cancel installation
```

3. **Press `1` (or just press Enter)** to proceed with the default installation
4. Wait for the download to complete (usually 1-2 minutes)
5. You'll see: `Rust is installed now. Great!`

### Step 3: Prerequisites — Visual C++ Build Tools

Rust on Windows needs the **Microsoft C++ Build Tools**. If you don't have them:

1. The installer will tell you if they're missing
2. Download from: [Visual Studio Build Tools](https://visualstudio.microsoft.com/visual-cpp-build-tools/)
3. In the installer, select **"Desktop development with C++"**
4. Install it (this is the biggest download, ~1-2 GB)

> **Why does Rust need C++ tools?** Rust links against the C standard library, and the Windows linker comes with Visual C++. This is a one-time setup.

#### What is a Linker and Why Does Rust Need One?

A **linker** is a program that combines multiple compiled files into a single executable. Here's why it's necessary:

When you write `println!("Hello")`, the actual "print to screen" code lives in the operating system's C standard library (`libc` on Linux/macOS, `msvcrt` on Windows). Your compiled Rust code contains a **reference** to this function, but doesn't include the actual implementation.

The linker resolves these references:

```
Your code (compiled):        System libraries:
┌─────────────────┐         ┌──────────────────────┐
│ main:           │         │ write():              │
│   call write()──┼────────→│   [actual code to     │
│                 │  linker │    write to screen]    │
└─────────────────┘ connects└──────────────────────┘
```

Without a linker, your program would be full of unresolved references — calls to functions that exist somewhere else. The linker finds those functions and wires everything together into a single, self-contained executable file.

On Windows, the linker comes bundled with Visual C++ Build Tools (`link.exe`). On Linux, it's part of `binutils` (`ld`). On macOS, it comes with Xcode Command Line Tools.

### Step 4: Restart Your Terminal

Close your terminal (PowerShell, Command Prompt, or Windows Terminal) and open a new one. This ensures the PATH is updated.

---

## Installing on macOS

### Step 1: Open Terminal

Press `Cmd + Space`, type "Terminal", press Enter.

### Step 2: Run the Install Command

Copy and paste this command:

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

Let's break down what this command does:

```
curl                          → Download a file from the internet
  --proto '=https'            → Only use HTTPS (secure connection)
  --tlsv1.2                   → Use TLS 1.2 encryption (secure)
  -sSf                        → Silent mode, show errors, fail on HTTP errors
  https://sh.rustup.rs        → The URL of the Rust install script
  | sh                        → Pipe the downloaded script to the shell to run it
```

### Step 3: Choose the Default Installation

You'll see the same options as Windows. Press `1` or Enter for the default installation.

### Step 4: Configure Your Shell

After installation, run:

```bash
source $HOME/.cargo/env
```

This adds Rust to your current terminal session. It's automatically added for future sessions.

> **Note:** If you use `zsh` (default on modern macOS), the installer adds the path to `~/.zshenv`. If you use `bash`, it adds to `~/.bash_profile`.

### Prerequisites

macOS needs Xcode Command Line Tools:

```bash
xcode-select --install
```

If you already have them, this will say so. If not, it'll install them (a few minutes).

---

## Installing on Linux

### Step 1: Open a Terminal

Use your distribution's terminal emulator.

### Step 2: Run the Install Command

Same command as macOS:

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

### Step 3: Default Installation

Press `1` or Enter.

### Step 4: Configure Your Shell

```bash
source $HOME/.cargo/env
```

### Prerequisites

You need a C compiler and linker. Install them based on your distro:

**Ubuntu / Debian:**
```bash
sudo apt update
sudo apt install build-essential
```

**Fedora:**
```bash
sudo dnf groupinstall "Development Tools"
```

**Arch Linux:**
```bash
sudo pacman -S base-devel
```

---

## Verifying Your Installation

Open a **new** terminal window and run these commands:

### Check the Rust Compiler

```bash
rustc --version
```

You should see something like:

```
rustc 1.82.0 (f6e511eec 2024-10-15)
```

The version number will be different — that's fine. As long as you see a version, it works!

### Check Cargo

```bash
cargo --version
```

Output:
```
cargo 1.82.0 (8f40fc59f 2024-08-21)
```

### Check Rustup

```bash
rustup --version
```

Output:
```
rustup 1.27.1 (54dd3d00f 2024-04-24)
```

### If Any of These Fail

See the [Troubleshooting](#troubleshooting) section below.

---

## Understanding What Got Installed

Let's look at what's on your computer now:

### Directory Structure

```
~/.rustup/                     ← Rustup home (toolchains live here)
  └── toolchains/
        └── stable-x86_64-.../  ← The stable toolchain
              ├── bin/
              │     ├── rustc    ← The compiler
              │     ├── rustdoc  ← Documentation generator
              │     └── ...
              └── lib/           ← Standard library

~/.cargo/                      ← Cargo home
  ├── bin/                     ← Your installed tools go here
  │     ├── cargo              ← The package manager
  │     ├── rustc              ← Proxy that calls the real rustc
  │     ├── rustup             ← The toolchain manager
  │     ├── rustfmt            ← Code formatter
  │     └── clippy-driver      ← Linter
  └── registry/                ← Downloaded crate source code (later)
```

### How the PATH Works

When you type `rustc`, here's what happens:

```
1. Your shell looks at your PATH environment variable
2. It finds ~/.cargo/bin/rustc
3. This is actually a "proxy" that asks rustup: "Which toolchain should I use?"
4. Rustup says "stable" (the default)
5. The proxy runs ~/.rustup/toolchains/stable-.../bin/rustc
6. Your code gets compiled!
```

This indirection lets you easily switch between Rust versions.

---

## Updating Rust

Rust releases a new stable version **every 6 weeks**. To update:

```bash
rustup update
```

That's it! This updates all installed toolchains (stable, beta, nightly).

### Rust Release Channels

| Channel | Description | Use Case |
|---------|-------------|----------|
| **stable** | Official releases, battle-tested | Use this! (default) |
| **beta** | Next release candidate, for testing | Library authors testing compatibility |
| **nightly** | Latest development, may break | Experimental features, some special crates |

You'll use **stable** for this entire tutorial series (and for most of your Rust career).

---

## Uninstalling Rust

If you ever need to uninstall:

```bash
rustup self uninstall
```

This removes everything: rustup, cargo, rustc, all toolchains. Clean uninstall.

---

## Setting Up Your Editor

You need a code editor with Rust support. Here are the best options:

### Option 1: Visual Studio Code (Recommended for Beginners)

1. Download [VS Code](https://code.visualstudio.com/)
2. Install the **rust-analyzer** extension:
   - Open VS Code
   - Press `Ctrl+Shift+X` (or `Cmd+Shift+X` on macOS)
   - Search for "rust-analyzer"
   - Click "Install"

**What rust-analyzer gives you:**
- Syntax highlighting (code is colorful and readable)
- Autocomplete (suggests functions, methods, types as you type)
- Inline type hints (shows types even when you don't write them)
- Error highlighting (red squiggles for mistakes — before you compile!)
- Go to definition (Ctrl+Click on any function/type to see its source)
- Hover documentation (hover over anything to see its docs)
- Code actions (automatic fixes for common issues)

### Option 2: RustRover (JetBrains)

If you prefer JetBrains IDEs:
1. Download [RustRover](https://www.jetbrains.com/rust/) (free for non-commercial use)
2. Rust support is built in — no extensions needed

### Option 3: Neovim / Helix / Zed

For more advanced users:
- **Neovim**: Use `rust-tools.nvim` or `rustaceanvim` plugin
- **Helix**: Built-in Rust support via LSP
- **Zed**: Built-in Rust support, very fast

### Recommended VS Code Settings for Rust

After installing rust-analyzer, add these to your VS Code `settings.json` (press `Ctrl+Shift+P` → "Open User Settings JSON"):

```json
{
    "rust-analyzer.check.command": "clippy",
    "editor.formatOnSave": true,
    "[rust]": {
        "editor.defaultFormatter": "rust-lang.rust-analyzer"
    }
}
```

This gives you:
- **Clippy** checks on save (better warnings than default)
- **Auto-format** on save (consistent code style)
- **rust-analyzer** as the default formatter for Rust files

---

## Troubleshooting

### "rustc is not recognized" / "command not found"

**Cause:** PATH isn't set up correctly.

**Fix on Windows:**
1. Open a NEW terminal (not the same one you installed in)
2. If still failing, check: System Settings → Environment Variables → PATH should contain `C:\Users\YourName\.cargo\bin`

**Fix on macOS/Linux:**
```bash
source $HOME/.cargo/env
```

If that fixes it temporarily but it doesn't persist, add this line to your shell config:

```bash
# For bash → add to ~/.bashrc or ~/.bash_profile:
source "$HOME/.cargo/env"

# For zsh → add to ~/.zshrc:
source "$HOME/.cargo/env"
```

### "linker 'cc' not found" (Linux)

**Fix:** Install build tools:
```bash
# Ubuntu/Debian
sudo apt install build-essential

# Fedora
sudo dnf groupinstall "Development Tools"
```

### "MSVC targets depend on the MSVC linker" (Windows)

**Fix:** Install Visual C++ Build Tools as described in the Windows section above.

### "SSL certificate problem" or curl errors

**Fix:** Update your system's CA certificates:
```bash
# Ubuntu/Debian
sudo apt install ca-certificates
```

Or download `rustup-init` directly from [https://rustup.rs](https://rustup.rs) instead of using curl.

---

## Summary

| Step | Command |
|------|---------|
| **Install Rust** | `curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs \| sh` (or download installer on Windows) |
| **Verify** | `rustc --version && cargo --version` |
| **Update** | `rustup update` |
| **Uninstall** | `rustup self uninstall` |
| **Editor** | VS Code + rust-analyzer extension |

### What You Installed

- **rustc** — the compiler
- **cargo** — the package manager and build tool
- **rustup** — the toolchain manager
- **rustfmt** — code formatter
- **clippy** — linter
- **rust-std** — the standard library
- **rust-docs** — offline documentation

### Key Takeaway

> Installing Rust is a one-command process. Updating is one command. Uninstalling is one command. Rust's tooling is designed to be painless.

---

## What's Next?

Rust is installed. Your editor is ready. Let's write some code!

**Next Tutorial:** [Hello, World! →](./03-hello-world.md)

---

<p align="center">
  <i>Tutorial 2 of 6 — Stage 1: The Starting Line</i>
</p>
