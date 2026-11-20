# Guide 01: Dev environment, part 1 (toolchain, Git/SSH, VS Code)

What I set up, **what each piece is for**, and how to redo it from scratch on a fresh Ubuntu machine. It's written for someone new to Linux and C++ development, which is me.

Machine: **Ubuntu 26.04 ("Resolute")** in a UTM/QEMU virtual machine on an Apple Silicon Mac, so an **ARM (arm64)** processor. The arm64 part matters in a few places, marked ⚠️. (I first thought this was 22.04; the package addresses saying `resolute` showed otherwise. Check with `lsb_release -a`.)

---

## 0. The big picture

The flight controller (or SITL on a PC) can't run `.cpp` files. Those are text for humans. A processor only runs machine code, which is what's in the `.hex` / `.apj` files I flash. Everything installed here either **turns text into machine code** or **helps me work on that text**.

```
                 Build system (waf / cmake + ninja)
                 decides what to compile, in what order, with which settings
                        │                    │
                        ▼                    ▼
Source code ──▶ Compiler (gcc/clang) ──▶ Linker ──▶ Program
.cpp / .h       one file → one .o       joins all    SITL binary,
                     ▲                  the .o files  or firmware
                     │                                    ▲
                 ccache                                   │
             (reuses past results)                       gdb
                                                 (pause and inspect)
 ▲
 └── clangd: lets the editor understand the code (jump to definition, errors as I type)
```

| Piece | One-line job | When I'll notice it |
|---|---|---|
| **Compiler** (gcc, clang) | Translates one `.cpp` file into machine code (an `.o` object file) | Compile errors: typos, wrong types |
| **Linker** | Joins hundreds of `.o` files into one program and connects calls between them | "undefined reference" errors: a missing piece |
| **Build system** (waf, CMake + Ninja) | Works out which files to compile, with which flags, and only rebuilds what changed | `./waf copter`, `cmake --build` |
| **ccache** | Remembers past compiles; same file + same flags → instant result | Second ArduPilot build takes minutes instead of 15+ |
| **gdb** (debugger) | Runs a program, stops it on any line, shows every variable | Finding *why* SITL does something odd, instead of guessing from logs |
| **clangd** | Reads code the way the compiler does, so VS Code can navigate it | Ctrl+click to jump to a function; red squiggles while typing |
| **Git** | Version history of my code; sync with GitHub | Commits, pushes, PRs |

Two build systems, because two worlds:
- **waf** is ArduPilot's own build system and comes with ArduPilot. `./waf configure --board sitl` sets the target; `./waf copter` builds it.
- **CMake + Ninja** is used by almost everything else: OpenVINS, ROS 2 packages, my own C++ projects. CMake reads `CMakeLists.txt` and writes a build plan, then Ninja carries it out quickly.

---

## 1. Terminal basics used below

| Thing | Meaning |
|---|---|
| `sudo <command>` | Run as administrator (asks for my password). Needed to install software. |
| `apt` | Ubuntu's app store for the terminal. `apt update` refreshes the list, `apt install x` installs x. |
| `~` | My home folder, `/home/<username>`. |
| `.something` | A file or folder starting with a dot is hidden. `ls -a` shows it. |
| `cat file` | Print a file to the screen. |
| `mv a b` | Move or rename a to b. |
| `chmod 600 file` | Set permissions. 600 = only I can read and write; 700 on a folder = only I can enter it. |
| `# text` | A comment. The shell ignores everything after `#`. |

---

## 2. Install the C++ toolchain

```bash
sudo apt update
sudo apt install -y build-essential git cmake ninja-build gdb \
    clang clangd clang-format pkg-config ccache bear
```

| Package | What it gives me |
|---|---|
| `build-essential` | gcc / g++ (the compiler ArduPilot uses by default) and make |
| `git` | Version control |
| `cmake`, `ninja-build` | Build system for everything except ArduPilot |
| `gdb` | The debugger |
| `clang`, `clangd` | A second compiler, and the editor helper built on it |
| `clang-format` | Auto-formats code for my own projects (ArduPilot has its own style; don't run it there) |
| `pkg-config` | Helps builds find installed libraries |
| `ccache` | The compile cache |
| `bear` | Records compile commands for builds that aren't CMake (see section 6) |

Check:
```bash
gcc --version && cmake --version && gdb --version && clangd --version
```
Any version number means it's installed. Ubuntu 26.04 ships recent versions of all of these, which is fine for everything in this plan.

---

## 3. Git identity + SSH key for GitHub

### Identity
Every commit is stamped with a name and email:
```bash
git config --global user.name "Elad"
git config --global user.email "<email on my GitHub account>"
git config --global init.defaultBranch main
```

### SSH key: how GitHub knows it's me
An SSH key is a **pair** of files:

| File | Role | Who gets it |
|---|---|---|
| `~/.ssh/id_ed25519` | **Private key**, the real key | Nobody. Never leaves this machine. |
| `~/.ssh/id_ed25519.pub` | **Public key**, the padlock | GitHub (and anywhere I want to log in) |

GitHub attaches my padlock to my account. When I push, my computer proves it holds the matching private key, without ever sending it. No password is involved.

```bash
ssh-keygen -t ed25519 -C "<my email>"   # press Enter at every prompt to accept the defaults
cat ~/.ssh/id_ed25519.pub                # print the public key, then copy the whole line
```
Then on GitHub: profile picture → **Settings → SSH and GPG keys → New SSH key**, paste, save.

Test:
```bash
ssh -T git@github.com
# → "Hi elb136! You've successfully authenticated, but GitHub does not provide shell access."
```
"Does not provide shell access" is normal and means it worked.

**The first time**, SSH asks "authenticity of host 'github.com' can't be established… continue?". Answer `yes`. GitHub's real ED25519 fingerprint is `SHA256:+DiY3wvvV6TuJJhbpZisF/zLDA0zPMSvHdkr4UvCOqU`.

### ⚠️ What went wrong for me, and the fix
At the "Enter file in which to save the key" prompt I typed a name (`git_key`) instead of pressing Enter. The key was saved as `~/git_key` and `~/git_key.pub`. SSH only checks `~/.ssh/` for standard names, so GitHub answered `Permission denied (publickey)`.

Fix: move the files to the standard place and name, and lock down the permissions (SSH ignores a private key that others could read):
```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
mv ~/git_key     ~/.ssh/id_ed25519
mv ~/git_key.pub ~/.ssh/id_ed25519.pub
chmod 600 ~/.ssh/id_ed25519
```
Still denied? `ssh -vT git@github.com` shows which key files SSH tried.

**Rule:** only ever share the `.pub` file. A file starting with `-----BEGIN OPENSSH PRIVATE KEY-----` is the private key. Never paste it anywhere.

---

## 4. VS Code

### ⚠️ Install on arm64
`sudo snap install code --classic` fails on ARM ("not available … for this architecture (arm64)"). Install from Microsoft's own package repository instead:
```bash
sudo apt install -y wget gpg apt-transport-https
wget -qO- https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor > packages.microsoft.gpg
sudo install -D -o root -g root -m 644 packages.microsoft.gpg /etc/apt/keyrings/packages.microsoft.gpg
echo "deb [arch=amd64,arm64,armhf signed-by=/etc/apt/keyrings/packages.microsoft.gpg] https://packages.microsoft.com/repos/code stable main" | sudo tee /etc/apt/sources.list.d/vscode.list > /dev/null
rm -f packages.microsoft.gpg
sudo apt update && sudo apt install -y code
```
What that does:
1. Downloads Microsoft's signing key, so Ubuntu can verify the packages are genuine.
2. Adds Microsoft's VS Code repository to Ubuntu's list of package sources.
3. Installs through `apt`, which picks the arm64 build and keeps it updated.

On an Intel/AMD machine, the snap command works fine.

Option for later: run VS Code on the host computer and connect to the VM with the **Remote - SSH** extension. The editor runs at full speed and builds still run in the VM.

### Extensions
```bash
code --install-extension llvm-vs-code-extensions.vscode-clangd   # code understanding
code --install-extension ms-vscode.cpptools                      # debugger integration
code --install-extension ms-vscode.cmake-tools                   # build/run CMake projects from VS Code
code --install-extension ms-python.python                        # for pymavlink / analysis scripts
```

### Settings
Ctrl+Shift+P → "Preferences: Open User Settings (JSON)":
```json
{
  "C_Cpp.intelliSenseEngine": "disabled",
  "clangd.arguments": ["--background-index", "--clang-tidy", "--header-insertion=never"],
  "cmake.configureSettings": { "CMAKE_EXPORT_COMPILE_COMMANDS": "ON" },
  "editor.formatOnSave": false
}
```
| Line | Why |
|---|---|
| `intelliSenseEngine: disabled` | The Microsoft extension and clangd both try to understand the code and conflict. clangd does that job; Microsoft's extension is kept only for debugging. |
| `clangd.arguments` | Index the whole project in the background, show extra code-quality hints, and don't auto-insert `#include` lines. |
| `CMAKE_EXPORT_COMPILE_COMMANDS: ON` | Makes CMake write `compile_commands.json` (section 6). |
| `formatOnSave: false` | Auto-formatting would rewrite ArduPilot files in my own style and fill PRs with noise. |

---

## 5. Hello-world check (proves the whole chain works)

```bash
mkdir -p ~/scratch/hello && cd ~/scratch/hello
cat > main.cpp <<'EOF'
#include <iostream>
#include <vector>
int main() {
    std::vector<double> v{1.0, 2.0, 3.0};
    double sum = 0;
    for (double x : v) sum += x;
    std::cout << "sum = " << sum << "\n";
}
EOF
cat > CMakeLists.txt <<'EOF'
cmake_minimum_required(VERSION 3.16)
project(hello CXX)
set(CMAKE_CXX_STANDARD 17)
add_executable(hello main.cpp)
EOF
code .
```
(`cat > file <<'EOF' … EOF` writes everything between the two `EOF` lines into the file.)

| Step in VS Code | What it proved |
|---|---|
| Pick the GCC kit | CMake Tools found the compiler |
| F7 builds | Build system → compiler → linker all work |
| Breakpoint on `sum += x`, then run under the debugger, watching `sum` | gdb works and is wired into VS Code |
| Hover over `std::vector` shows its documentation | clangd is reading the code correctly |

Result on 2026-11-19: ✅ all four.

---

## 6. `compile_commands.json`: why clangd needs it

To understand a file, clangd must know how it was compiled: which folders hold the `#include` files, and which settings (board, features) were on. The compiler gets these from the build system; clangd doesn't, unless I give it this file.

`compile_commands.json` = a list of "this file was compiled with this exact command". I never write it by hand:

| Project type | How the file gets made |
|---|---|
| CMake (OpenVINS, ROS 2, my projects) | Automatic, from the VS Code setting above |
| ArduPilot (waf) | After building, check `build/sitl/compile_commands.json`. If it's missing, build with `bear -- ./waf copter`; bear watches the build and writes the file. |

clangd looks for the file in the **project root** (the top folder). If it sits in `build/`, create a shortcut (symlink) in the root:
```bash
ln -s build/compile_commands.json .    # "make a shortcut to build/compile_commands.json, here"
```
Symptom of a missing file: red squiggles on every `#include`, and Ctrl+click goes nowhere.

---

## 7. ⚠️ Known limits of this machine (arm64 VM)

- **Works:** ArduPilot SITL, the C++ tools, ROS 2 Humble, OpenVINS, datasets.
- **Possible snags:** install scripts or prebuilt binaries that assume Intel/AMD. The error usually says "architecture", and there's usually an ARM route like the VS Code fix above.
- **The real limit is Gazebo.** A QEMU VM usually has no 3D graphics acceleration, so Gazebo may be very slow or fail to start. Decide before Phase 3: run Gazebo on the host, use a better-accelerated VM, or a cloud machine.

---

## Next: guide 02

Clone ArduPilot, install its prerequisites, build SITL, get `compile_commands.json`, first SITL flight, then SITL under gdb.
