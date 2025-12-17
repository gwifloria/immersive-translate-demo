# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **Chrome Manifest V3 browser extension** for "Immersive Translate" - a web page translation tool. The repository contains compiled/bundled extension code (not source code), so there is no build system or development workflow.

## Extension Architecture

### Core Components

- **`background.js`** - Service worker that handles extension background tasks, API communications, and cross-tab coordination
- **`content_script.js`** - Injected into all web pages to handle page translation, DOM manipulation, and UI injection
- **`popup.js` / `popup.html`** - Extension popup UI when clicking the toolbar icon
- **`options.js` / `options.html`** - Extension settings/configuration page
- **`side-panel.js` / `side-panel.html`** - Chrome side panel interface
- **`offscreen.js` / `offscreen.html`** - Offscreen document for background processing

### Feature Modules

- **`video-subtitle/inject.js`** - Hooks into video players (YouTube, Netflix, Udemy, etc.) to translate subtitles
- **`image/inject.js`** - Image translation via OCR (uses Tesseract)
- **`browser-bridge/inject.js`** - Bridge for cross-context communication
- **`pdf/`** - PDF translation support
- **`tesseract/`** - OCR library for image text extraction

### Configuration

- **`manifest.json`** - Extension manifest defining permissions, content scripts, commands, etc.
- **`default_config.json`** - Default extension configuration
- **`default_config.content.json`** - Content-related default configuration
- **`locales.json`** - All translation strings
- **`_locales/`** - Chrome i18n localization files

## Key Features

- Page translation (multiple services: Google, DeepL, OpenAI, Claude, Bing, etc.)
- Video subtitle translation (YouTube, Netflix, Udemy, Hulu, Disney+, etc.)
- Image/OCR translation
- Input box translation
- PDF translation
- Side panel UI
- Keyboard shortcuts (Alt+A for translate page, Alt+W for whole page, Alt+S for side panel)

## Important Notes

- All JavaScript files are minified/bundled - they are not source code
- The extension uses Chrome's Manifest V3 APIs (service workers, declarativeNetRequest, etc.)
- Version: 1.21.7 (check `manifest.json` for current version)
