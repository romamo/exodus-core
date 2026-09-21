# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

`exodus_core` is the core library behind [εxodus Privacy](https://exodus-privacy.eu.org): static analysis of Android APKs to detect embedded trackers, extract metadata (certificates, permissions, icon perceptual hash), plus a small HTTP connector to an εxodus server instance. It is published to PyPI as `exodus-core`.

## Commands

Tests require `dexdump` on `PATH` (it lives in `exodus_core/dexdump/`).

```shell
export PATH="$PATH:$PWD/exodus_core/dexdump/"
python -m unittest discover -v -s exodus_core -p "test_*.py"     # all tests
python -m unittest exodus_core.analysis.test_exodus_analyze.TestExodus.test_trackers_list   # single test
flake8                                                           # lint (config in .flake8)
```

Docker is the reference environment (Debian, `dexdump` from apt, `PATH` preconfigured):

```shell
docker build -t exodus-core .
docker run -it --rm exodus-core python -m unittest discover -s exodus_core -p "test_*.py"
```

Note: `setup.py` refuses to run on `darwin`/`win32`. On macOS, use Docker for anything that runs the full analysis pipeline. CI runs on Python 3.10 and 3.14 (oldest and newest supported).

Sample APKs for manual testing live in `apks/` (whatsapp, nextcloud, hsbc, etc.). Several tests in `test_exodus_analyze.py` reference APK paths and are effectively fixtures/examples.

## Architecture

Three independent surfaces under `exodus_core/`:

- **`analysis/static_analysis.py`** — the heart of the library. `StaticAnalysis(apk_path)` wraps an androguard `APK` and exposes everything downstream needs: `load_trackers_signatures()` (fetches signatures live from `https://reports.exodus-privacy.eu.org/api/trackers` and compiles each `code_signature` regex), `get_embedded_classes()`, `detect_trackers()`, `get_certificates()`, `get_permissions()`, `get_icon_phash()`, `get_application_universal_id()`. `apk_path` may be a filesystem path (str) or an in-memory buffer (`.getvalue()` + `raw=True`).
- **`analysis/apk_signature.py`** — `ApkSignature` is a thin aggregator: it instantiates `StaticAnalysis` and eagerly pulls a fixed set of fields (size, sha256, handle, versions, icon pHash, permissions, app UID, certificates) into one printable value object. Excluded from flake8 (see `.flake8`).
- **`helper/connector.py`** — `ExodusConnector(host, report_info_uri)`: token login, download APK, upload pcap against an εxodus server. Unrelated to the analysis code.

`analysis/certificate.py` + `analysis/utils.py` are a separate, older certificate-parsing path adapted from AndroidObservatory (shells out via `check_output`); the primary certificate flow used by `StaticAnalysis.get_certificates()` is the inline `Certificate` class in `static_analysis.py` built on `cryptography`.

### Key behaviors to know before editing

- **Class extraction shells out to `dexdump`.** `_get_embedded_classes()` unzips DEX files and parses `dexdump` text output with regex. It recurses into nested APKs with a depth-10 zip-bomb guard, and `sys.exit(1)` (not an exception) if `dexdump` is missing. `ExtractionError` is raised on bad zips / dexdump failure.
- **Tracker detection** = match compiled signature regexes against the extracted class list. Signatures are loaded over the network, so tests and callers must call `load_trackers_signatures()` first, and that call needs connectivity.
- **`clean_embedded_classes()`** applies hand-tuned heuristics (path length, known framework prefixes like `java`/`androidx`/`kotlin`/`okhttp3`) to strip obfuscated/noise classes. Changing these thresholds changes detection results.
- **`get_details_from_gplaycli()` is intentionally stubbed** to return `None` (Google Play token flow is broken upstream); the real implementation is commented out. Don't wire it back up without a working token source.
- Icon similarity uses `dhash` perceptual hashing (`PHASH_SIZE = 8` in the module; tests use 16).

## Conventions

- Dependency ranges in `setup.py` `install_requires` are authoritative for release; `requirements.txt` mirrors them for local/dev install. Keep them as lower bounds (plus a major-version cap where APIs are sensitive), not exact pins, so downstream apps can resolve alongside them. `androguard>=4.1.1,<5` is the analysis backbone and the APIs used (`androguard.core.apk.APK`, `androguard.core.axml`) are version-sensitive.
- `flake8` ignores `E501` (line length) and `W605`; `apk_signature.py` is fully excluded.
- Version lives in `setup.py` (`version=`). Releases are tag-driven: pushing a `v*` tag triggers CI to build an sdist and publish to PyPI.
