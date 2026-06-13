# PySide6 6.11.1 Full HTML Documentation Build Guide
## Debian Trixie (13.x) — x86_64 — Verified Working

---

## Overview

This guide builds the complete PySide6 6.11.1 HTML documentation with working
cross-module links (QtCore, QtGui, QtWidgets etc.) on Debian Trixie using Python
3.12 via pyenv. System Python 3.13.5 is bypassed entirely.

**Approximate build time:** 30–60 minutes  
**Disk space required:** ~15 GB  
**Output location:** `~/pyside6-docs/`

---

## Phase 1 — System Prerequisites

### 1.1 Verify baseline

```bash
cat /etc/debian_version   # should be 13.x
uname -m                  # should be x86_64
python3 --version         # will show 3.13.x — we will bypass this
```

### 1.2 Install system build tools and C libraries

These are build tools and C shared libraries — they cannot go in a venv.

```bash
sudo apt update && sudo apt install -y \
    build-essential git cmake ninja-build \
    libreadline-dev libncurses-dev libbz2-dev \
    libsqlite3-dev liblzma-dev zlib1g-dev tk-dev \
    libclang-dev libxml2-dev libxslt1-dev libssl-dev \
    graphviz python3-dev python3-pip python3-venv \
    wget curl p7zip-full pkg-config patchelf \
    libfontconfig1-dev libfreetype6-dev \
    libx11-dev libxext-dev libxfixes-dev libxi-dev \
    libxrender-dev libxcb1-dev libx11-xcb-dev \
    libxcb-glx0-dev libxcb-keysyms1-dev libxcb-image0-dev \
    libxcb-shm0-dev libxcb-icccm4-dev libxcb-sync-dev \
    libxcb-xfixes0-dev libxcb-shape0-dev libxcb-randr0-dev \
    libxcb-render-util0-dev libxcb-util-dev \
    libxcb-xinerama0-dev libxcb-xkb-dev \
    libxkbcommon-dev libxkbcommon-x11-dev libglu1-mesa-dev
```

Verify:
```bash
cmake --version && ninja --version && git --version && dot -V
```

---

## Phase 2 — Python 3.12 via pyenv

System Python 3.13 is not suitable — use pyenv to install 3.12.

### 2.1 Install pyenv

```bash
curl https://pyenv.run | bash
```

Add to shell (and to `~/.bashrc` permanently):
```bash
export PYENV_ROOT="$HOME/.pyenv"
export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init -)"
eval "$(pyenv virtualenv-init -)"

cat >> ~/.bashrc << 'EOF'

# pyenv
export PYENV_ROOT="$HOME/.pyenv"
export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init -)"
eval "$(pyenv virtualenv-init -)"
EOF
```

### 2.2 Install Python 3.12.9

```bash
pyenv install 3.12.9
pyenv global 3.12.9
python --version   # must show Python 3.12.9
which python       # must show ~/.pyenv/shims/python
```

---

## Phase 3 — Qt 6.11.1 via Qt Online Installer

### 3.1 Download and run the installer

```bash
cd ~/Downloads
wget https://d13lb3tujbc8s0.cloudfront.net/onlineinstallers/qt-online-installer-linux-x64-4.8.1.run
chmod +x qt-online-installer-linux-x64-4.8.1.run
./qt-online-installer-linux-x64-4.8.1.run
```

Requires a free Qt account at https://login.qt.io/register

### 3.2 Select components

Under Qt 6.11.1, select **only**:
- ✅ `Desktop` (this is gcc 64-bit on Linux)
- ✅ `Sources`
- ❌ Everything else (Qt Doc, Debug Info, CMake, Ninja — skip all)

Default install path `~/Qt` is correct.

> If you get a CDN hash error during download, simply rerun the installer.

### 3.3 Verify

```bash
ls ~/Qt/6.11.1/                             # should show gcc_64 Src
~/Qt/6.11.1/gcc_64/bin/qtpaths --version   # qtpaths 2.0
~/Qt/6.11.1/gcc_64/bin/qdoc --version      # qdoc 6.11.1
```

---

## Phase 4 — Qt's Prebuilt libclang 18

Must use Qt's bundled libclang, not the system one.

### 4.1 Download and extract

```bash
mkdir -p ~/libclang && cd ~/libclang
wget https://download.qt.io/development_releases/prebuilt/libclang/libclang-release_18.1.7-based-linux-Ubuntu22.04-gcc11.4-x86_64.7z
7z x libclang-release_18.1.7-based-linux-Ubuntu22.04-gcc11.4-x86_64.7z
```

> The 3 "Dangerous link" errors from 7z are harmless — they are clang++ symlinks
> we don't need.

> The Ubuntu 22.04 label is fine on Trixie — glibc is forwards compatible.

### 4.2 Set environment variable

```bash
export LLVM_INSTALL_DIR=$HOME/libclang/libclang
echo 'export LLVM_INSTALL_DIR=$HOME/libclang/libclang' >> ~/.bashrc
```

Verify:
```bash
ls $LLVM_INSTALL_DIR/lib/libclang.so   # must exist
```

---

## Phase 5 — Clone pyside-setup

```bash
cd ~
git clone https://code.qt.io/pyside/pyside-setup.git
cd pyside-setup
git checkout v6.11.1
git submodule update --init --recursive
```

Verify:
```bash
git describe --tags        # v6.11.1
ls ~/pyside-setup/sources/ # pyside6 pyside-tools shiboken6 shiboken6_generator
```

> The commit message "Pin Qt5 sha1..." is misleading but harmless — the tag
> v6.11.1 is correct and the code is Qt6.

---

## Phase 6 — Python Virtual Environment

Keep the venv outside the source tree.

### 6.1 Create and activate

```bash
python -m venv ~/.venvs/pyside6docs
source ~/.venvs/pyside6docs/bin/activate
python --version   # must show 3.12.9
```

### 6.2 Install all required packages (pinned versions)

These exact versions are verified to work together without conflicts:

```bash
pip install --upgrade pip setuptools wheel

pip install \
    alabaster==0.7.16 \
    apeye==1.4.1 \
    apeye-core==1.1.5 \
    autodocsumm==0.2.15 \
    babel==2.18.0 \
    beautifulsoup4==4.14.3 \
    certifi==2026.5.20 \
    charset-normalizer==3.4.7 \
    dict2css==0.6.0 \
    docutils==0.18.1 \
    domdf_python_tools==3.10.0 \
    furo==2023.9.10 \
    graphviz==0.20.1 \
    imagesize==2.0.0 \
    Jinja2==3.1.6 \
    markdown-it-py==3.0.0 \
    MarkupSafe==3.0.3 \
    mdit-py-plugins==0.6.1 \
    myst-parser==2.0.0 \
    natsort==8.4.0 \
    packaging==24.2 \
    Pygments==2.16.1 \
    PyYAML==6.0.3 \
    requests==2.31.0 \
    roman-numerals==4.1.0 \
    six==1.17.0 \
    snowballstemmer==3.1.0 \
    Sphinx==7.2.6 \
    sphinx-autodoc-typehints==1.25.2 \
    sphinx-copybutton==0.5.2 \
    sphinx-design==0.7.0 \
    sphinx-jinja2-compat==0.4.1 \
    sphinx-prompt==1.10.2 \
    sphinx-reredirects==0.1.2 \
    sphinx-tabs==3.4.4 \
    sphinx-tags==0.3.1 \
    sphinx-toolbox==3.5.0 \
    sphinxcontrib-applehelp==2.0.0 \
    sphinxcontrib-devhelp==2.0.0 \
    sphinxcontrib-htmlhelp==2.1.0 \
    sphinxcontrib-jsmath==1.0.1 \
    sphinxcontrib-qthelp==2.0.0 \
    sphinxcontrib-serializinghtml==2.0.0
```

### 6.3 Verify critical versions

```bash
pip list | grep -E "^Sphinx |^docutils|^furo|^myst|^sphinx-tabs|^sphinx-autodoc"
```

Expected:
```
Sphinx                   7.2.6
docutils                 0.18.1
furo                     2023.9.10
myst-parser              2.0.0
sphinx-autodoc-typehints 1.25.2
sphinx-tabs              3.4.4
```

---

## Phase 7 — Set Environment Variables

```bash
export PATH="$HOME/Qt/6.11.1/gcc_64/bin:$PATH"
export QT_SRC_DIR="$HOME/Qt/6.11.1/Src/qtbase"

echo 'export PATH="$HOME/Qt/6.11.1/gcc_64/bin:$PATH"' >> ~/.bashrc
echo 'export QT_SRC_DIR="$HOME/Qt/6.11.1/Src/qtbase"' >> ~/.bashrc
```

### Full environment checklist — verify before building

```bash
echo "=== Python ===" && python --version && which python
echo "=== Qt ===" && qtpaths --version
echo "=== qdoc ===" && which qdoc
echo "=== libclang ===" && echo $LLVM_INSTALL_DIR && ls $LLVM_INSTALL_DIR/lib/libclang.so
echo "=== Qt Source ===" && ls $QT_SRC_DIR/CMakeLists.txt
echo "=== ninja ===" && ninja --version
echo "=== cmake ===" && cmake --version
echo "=== graphviz ===" && dot -V
echo "=== venv ===" && echo $VIRTUAL_ENV
```

All must return real values before proceeding.

---

## Phase 8 — Build PySide6

### 8.1 Recommended: run inside tmux

```bash
tmux new -s pyside_build
# Inside tmux, re-activate the environment:
source ~/.venvs/pyside6docs/bin/activate
export PATH="$HOME/Qt/6.11.1/gcc_64/bin:$PATH"
export QT_SRC_DIR="$HOME/Qt/6.11.1/Src/qtbase"
export LLVM_INSTALL_DIR=$HOME/libclang/libclang
```

Tmux tips:
- Detach without stopping build: `Ctrl+B` then `D`
- Reattach later: `tmux attach -t pyside_build`
- Scroll in tmux: `Ctrl+B` then `[`, then arrow keys, `q` to exit

Monitor log from a second terminal:
```bash
tail -f ~/pyside_build.log
```

### 8.2 Run the build

```bash
cd ~/pyside-setup

python setup.py install \
    --qtpaths=$HOME/Qt/6.11.1/gcc_64/bin/qtpaths \
    --qt-src-dir=$HOME/Qt/6.11.1/Src/qtbase \
    --build-docs \
    --ignore-git \
    --parallel=$(nproc) \
    2>&1 | tee ~/pyside_build.log
```

> `--parallel=$(nproc)` automatically uses all CPU threads.

This builds shiboken6, shiboken6_generator, and pyside6. Takes ~7 minutes.

### 8.3 Run ninja apidoc (generates the HTML)

```bash
cd ~/pyside-setup/build/pyside6docs/build/pyside6/
ninja apidoc -j$(nproc) 2>&1 | tee ~/ninja_apidoc.log
```

This runs 4 steps:
1. Copying docs
2. Generating release notes + example gallery + snippets
3. Running qdoc against Qt source (the longest step — ~30–50 min)
4. Running Sphinx to generate HTML

> The thousands of qdoc warnings in steps 2-3 are normal and harmless.
> "1954 warnings (222 known issues)" with exit code 0 means success.

---

## Phase 9 — Copy Output

```bash
cp -r ~/pyside-setup/build/pyside6docs/build/pyside6/doc/html ~/pyside6-docs
xdg-open ~/pyside6-docs/index.html
```

Qt module pages are at:
```
~/pyside6-docs/PySide6/QtCore/QObject.html
~/pyside6-docs/PySide6/QtWidgets/QWidget.html
# etc.
```

---

## Notes on Dependency Conflicts

Several pip packages pull in newer Sphinx/docutils as dependencies. If you
see Sphinx or docutils get upgraded during any pip install, immediately run:

```bash
pip install --force-reinstall sphinx==7.2.6 docutils==0.18.1
```

The following packages are the known culprits — their pinned safe versions
are already included in the install list above:
- `myst-parser` → pin to 2.0.0
- `sphinx-toolbox` → pin to 3.5.0
- `sphinx-autodoc-typehints` → pin to 1.25.2
- `sphinx-reredirects` → pin to 0.1.2
- `sphinx-tags` → pin to 0.3.1

Always verify after any pip install:
```bash
pip list | grep -E "^Sphinx |^docutils"
# Must show: Sphinx 7.2.6 and docutils 0.18.1
```

---

## Why These Choices

| Decision | Reason |
|---|---|
| Python 3.12 not 3.13 | Shiboken6 cross-references broken with 3.13 on Trixie |
| pyenv not system Python | Isolates from system 3.13 without root access |
| Qt's libclang not system | Shiboken requires exact libclang ABI match |
| Ubuntu 22.04 libclang on Trixie | glibc is forwards compatible — works fine |
| Sphinx 7.2.6 pinned | Newer versions break sphinx-tabs and furo |
| docutils 0.18.1 pinned | sphinx-tabs requires exactly ~0.18.0 |
| venv outside source tree | Prevents cmake from scanning venv files |
