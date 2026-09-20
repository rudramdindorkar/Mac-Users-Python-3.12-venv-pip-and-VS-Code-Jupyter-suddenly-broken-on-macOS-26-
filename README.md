# Mac-Users-Python-3.12-venv-pip-and-VS-Code-Jupyter-suddenly-broken-on-macOS-26-
# 🍎 Mac Users: Python 3.12 `venv`, `pip` and VS Code Jupyter suddenly broken on macOS 26? Here's what happened.

I recently spent around 20 minutes trying to create a simple Python 3.12 virtual environment on my Apple Silicon Mac.

My goal was straightforward:

```text
macOS 26.0
      ↓
Homebrew Python 3.12
      ↓
venv312
      ↓
ipykernel
      ↓
VS Code Jupyter
```

Instead:

```bash
python3.12 -m venv venv312
```

kept failing with:

```text
Error: Command [...] '-m', 'ensurepip', '--upgrade', '--default-pip'
returned non-zero exit status 1.
```

I initially thought Python 3.12, `venv`, or pip was broken.

It wasn't that simple.

---

## My environment

```text
Mac: Apple Silicon
macOS: 26.0
Python: 3.12.14
Installation: Homebrew
Use case: VS Code + Jupyter
```

Python itself worked:

```bash
python3.12 --version
```

```text
Python 3.12.14
```

Even pip worked globally:

```bash
python3.12 -m pip --version
```

```text
pip 26.2.1 ...
```

But creating a virtual environment failed.

---

## The clue that finally mattered

I checked:

```bash
python3.12 -c "import platform; print(platform.mac_ver())"
```

and got:

```text
('', ('', '', ''), '')
```

At the same time:

```bash
python3.12 -c "import sysconfig; print(sysconfig.get_platform())"
```

returned:

```text
macosx-26.0-arm64
```

So Python knew it was running on macOS 26 through `sysconfig`, but `platform.mac_ver()` was returning an empty version.

The next failure gave an even bigger clue.

Running pip produced:

```text
ValueError: invalid literal for int() with base 10: ''
```

inside pip's macOS `truststore` code.

---

# What was actually wrong?

The problem turned out to be related to **`pyexpat` and the system `libexpat` library**.

In my Homebrew Python installation, `pyexpat` was linked against:

```text
/usr/lib/libexpat.1.dylib
```

On macOS 26, this created a compatibility problem with the Homebrew Python build.

That problem then surfaced in places that seemed completely unrelated:

```text
pyexpat / library loading
        ↓
Python platform detection / SSL trust handling
        ↓
pip
        ↓
ensurepip
        ↓
python -m venv
```

So the error message made it look like:

> "`venv` is broken"

when the underlying issue was lower in the dependency chain.

---

# The fix

I installed Homebrew's own Expat library:

```bash
brew install expat
```

Then located Python's `pyexpat` extension:

```bash
PYEXPAT="$(/opt/homebrew/bin/python3.12 -c 'import importlib.util; print(importlib.util.find_spec("pyexpat").origin)')"
```

Checked its library dependencies:

```bash
otool -L "$PYEXPAT" | grep expat
```

Then redirected it from Apple's system Expat to Homebrew's Expat:

```bash
install_name_tool -change /usr/lib/libexpat.1.dylib "$(brew --prefix expat)/lib/libexpat.1.dylib" "$PYEXPAT"
```

Because the Mach-O file was modified, I re-signed it:

```bash
codesign --sign - --force "$PYEXPAT"
```

Then tested:

```bash
/opt/homebrew/bin/python3.12 -c "import pyexpat; print('pyexpat OK')"
```

Result:

```text
pyexpat OK
```

---

# Finally, `venv` worked

I recreated the environment:

```bash
rm -rf venv312
```

```bash
/opt/homebrew/bin/python3.12 -m venv ./venv312
```

Activated it:

```bash
source ./venv312/bin/activate
```

Verified:

```bash
python --version
```

```text
Python 3.12.14
```

Then:

```bash
python -m pip --version
```

worked normally.

---

# Getting it into VS Code Jupyter

I installed `ipykernel`:

```bash
python -m pip install ipykernel
```

Registered the kernel:

```bash
python -m ipykernel install --user --name=venv312 --display-name="Python 3.12 (venv312)"
```

Then in VS Code:

```text
Notebook
   ↓
Select Kernel
   ↓
Python 3.12 (venv312)
```

And it worked.

---

# A quick diagnostic for other Mac users

Before reinstalling Python repeatedly, check these:

```bash
python3.12 --version
```

```bash
python3.12 -c "import platform; print(platform.mac_ver())"
```

```bash
python3.12 -c "import sysconfig; print(sysconfig.get_platform())"
```

```bash
python3.12 -m pip --version
```

Find `pyexpat`:

```bash
PYEXPAT="$(python3.12 -c 'import importlib.util; print(importlib.util.find_spec("pyexpat").origin)')"
```

Then:

```bash
otool -L "$PYEXPAT" | grep expat
```

If you're on macOS 26 and see the same combination of:

```text
platform.mac_ver() → ('', ('', '', ''), '')
```

and a `pyexpat`/`libexpat` linkage to the system library, you may be hitting the same class of issue.

---

# What I tried before finding the actual cause

I tried several things that looked reasonable:

* Recreating the virtual environment
* Creating it with `--without-pip`
* Using `get-pip.py`
* Reinstalling Python 3.12
* Trying to install `ipykernel` manually
* Using another Python environment

Those approaches didn't address the underlying library mismatch.

The important lesson was:

> **Don't assume that a `venv` failure means the venv is the problem.**

Sometimes `venv` is just where the underlying dependency problem becomes visible.

---

# Is this a known issue?

Yes.

This isn't a problem I discovered independently. Homebrew has an upstream issue documenting the `pyexpat`/`libexpat` incompatibility on macOS 26, and other developers have reported related failures involving Homebrew Python, `ensurepip`, pip, and virtual environments.

The relevant upstream references are:

* **Homebrew/homebrew-core #277330** — macOS 26 / `pyexpat` / `libexpat`
* **CPython #135675** — `platform.mac_ver()` behavior on macOS Tahoe

So this post is not meant to claim a new fix.

It's a **consolidated Mac + Python 3.12 + VS Code/Jupyter troubleshooting guide** showing the complete chain from the original error to the working kernel.

---

# Final takeaway

If you're on an Apple Silicon Mac running macOS 26 and suddenly see:

```text
python -m venv → ensurepip failed
```

don't immediately start reinstalling every Python version you have.

Check:

```text
Python version
↓
platform.mac_ver()
↓
pyexpat
↓
libexpat linkage
↓
pip / ensurepip
```

That diagnostic path can save a lot of time.


