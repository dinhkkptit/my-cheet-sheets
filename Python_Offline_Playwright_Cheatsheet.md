# Python Offline Installation and Chrome CDP Cheatsheet

Commands below are for Windows Command Prompt (CMD).

## 1. Check the offline machine

Python and pip must already be installed. `pip download` downloads packages, not Python itself.

```bat
python --version
python -m pip --version
python -c "import platform, struct; print(platform.system(), platform.machine()); print('Python bits:', struct.calcsize('P') * 8)"
where python
```

Match the target OS, Python major/minor version and architecture when downloading wheels.

## 2. Download packages with dependencies — online machine

Dependencies are downloaded automatically unless you specify `--no-deps`.

```bat
python -m pip download --only-binary=:all: --dest packages requests tzdata playwright
```

For graph and Office exports, include packages your script uses:

```bat
python -m pip download --only-binary=:all: --dest packages requests tzdata playwright matplotlib openpyxl python-pptx
```

`--only-binary=:all:` requires wheels, avoiding source builds that might need compilers and additional dependencies offline.

### Download from requirements.txt

Example file:

```text
requests
tzdata
playwright
```

```bat
python -m pip download --only-binary=:all: --dest packages -r requirements.txt
```

Use `package==VERSION` to pin a required version.

## 3. Download for a different Python version

Example: offline target is standard CPython 3.13 on Windows x64.

```bat
python -m pip download --dest packages313 --only-binary=:all: --platform win_amd64 --python-version 3.13 --implementation cp --abi cp313 requests tzdata playwright
```

| Target | Python option | ABI option | Platform |
|---|---|---|---|
| CPython 3.12 Windows x64 | `--python-version 3.12` | `--abi cp312` | `--platform win_amd64` |
| CPython 3.13 Windows x64 | `--python-version 3.13` | `--abi cp313` | `--platform win_amd64` |
| CPython 3.14 Windows x64 | `--python-version 3.14` | `--abi cp314` | `--platform win_amd64` |

These examples assume standard CPython, not free-threaded builds. A matching online environment is preferable for complex platform-dependent dependencies.

## 4. Install on the offline machine

Copy all downloaded wheels to `F:\pip`.

```bat
python -m pip install --no-index --find-links="F:\pip" requests tzdata playwright
```

Or copy requirements.txt too:

```bat
python -m pip install --no-index --find-links="F:\pip" -r "F:\requirements.txt"
```

- `--no-index`: do not search internet package indexes.
- `--find-links`: look in this folder for packages and dependencies.

### Install a specific .whl with local dependencies

Replace the filename with the actual wheel filename:

```bat
python -m pip install --no-index --find-links="F:\pip" "F:\pip\package_name.whl"
```

All required dependency wheels must be present in the folder.

### Optional virtual environment

```bat
python -m venv C:\venvs\automation
C:\venvs\automation\Scripts\activate.bat
python -m pip install --no-index --find-links="F:\pip" requests tzdata playwright
```

## 5. Wheel compatibility and troubleshooting

Example wheel:

```text
charset_normalizer-3.5.2-cp314-cp314-win_amd64.whl
```

| Tag | Meaning |
|---|---|
| `cp314` | CPython 3.14 |
| `cp313` | CPython 3.13 |
| `win_amd64` | Windows x64 |
| `py3-none-any` | Platform-independent Python 3 wheel; package Python-version requirements still apply |

A `cp314-cp314` wheel does not work with Python 3.13. Do not rename wheel files to bypass compatibility checks.

### Error: No matching distribution found

Check the interpreter and supported wheel tags:

```bat
python --version
python -m pip debug --verbose
```

Check that the folder contains every dependency and that its wheel tags match the offline interpreter. Compiled dependencies such as greenlet, numpy and matplotlib must also match. Download again into a fresh folder for the correct target. If compatibility matches, check pip version and package Python-version requirements.

## 6. Playwright package versus browser

| Usage | Separate Playwright browser download needed? |
|---|---|
| Attach to installed Chrome over CDP | No |
| Launch installed Chrome with `channel="chrome"` | No |
| Launch default Playwright Chromium | Yes |

`pip download playwright` includes Python dependencies but does not include browser binaries.

## 7. Start installed Chrome headless in the background

Adjust the executable path if Chrome is installed elsewhere:

```bat
start "" "C:\Program Files\Google\Chrome\Application\chrome.exe" --headless --remote-debugging-port=9222 --user-data-dir="C:\chrome-cdp-profile" about:blank
```

- `start ""`: launches without waiting; the empty string is the CMD window title.
- `--headless`: no visible Chrome window.
- `--remote-debugging-port=9222`: local CDP endpoint.
- `--user-data-dir`: separate automation profile.

Chrome continues running after the command returns. Reusing this profile retains cookies/settings. Do not run multiple instances with the same profile directory. Keep the debugging endpoint local.

Check CDP:

```bat
curl http://127.0.0.1:9222/json/version
```

## 8. Python CDP test — no internet required

Save as `test_cdp.py`:

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.connect_over_cdp("http://127.0.0.1:9222")
    context = browser.contexts[0]
    page = context.new_page()
    page.set_content("<h1>Offline Chrome CDP works!</h1>")
    page.screenshot(path="test.png")
    page.close()
```

```bat
python test_cdp.py
```

Headless mode is set when Chrome starts, not when Playwright connects.

## 9. Alternative: Playwright launches installed Chrome

No separate Chrome command or CDP port is needed for this pattern:

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch(channel="chrome", headless=True)
    page = browser.new_page()
    page.set_content("<h1>Installed Chrome works!</h1>")
    page.screenshot(path="test.png")
    browser.close()
```

## 10. Bundle Playwright Chromium for offline use

On an online machine matching the target OS/architecture, install the exact Playwright version used offline. Then:

```bat
set "PLAYWRIGHT_BROWSERS_PATH=C:\pw-browsers"
python -m playwright install chromium
```

Copy the complete `C:\pw-browsers` directory to the offline machine. Before running the script there:

```bat
set "PLAYWRIGHT_BROWSERS_PATH=C:\pw-browsers"
python your_script.py
```

The environment variable applies to the current CMD session. Browser binaries must match the Playwright version. Linux may also need separately provisioned system libraries.

## 11. Verify packages

```bat
python -m pip check
python -m pip show playwright
python -c "import requests, tzdata; from playwright.sync_api import sync_playwright; print('Imports OK')"
```

## Official documentation

- https://pip.pypa.io/en/stable/cli/pip_download/
- https://pip.pypa.io/en/stable/cli/pip_install/
- https://playwright.dev/python/docs/browsers
- https://playwright.dev/python/docs/api/class-browsertype#browser-type-connect-over-cdp
