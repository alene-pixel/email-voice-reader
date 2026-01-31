# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Email Voice Reader is a browser-based web application that lets users listen to their emails via text-to-speech and control the app with voice commands. The entire application is contained in a single `index.html` file.

This app was built collaboratively by Claude Cowork, Alene, and ChatGPT.

**Mobile-first**: This app is designed for mobile use. When making edits, always prioritize how the app will work on mobile devices.

## Deployment

The live application is hosted on GitHub. This local repository is a working copy. After making changes, push to GitHub for them to take effect.

## Running the Application

Open `index.html` directly in a browser.

The app requires internet access for:
- Google Sign-In (OAuth 2.0)
- Gmail API
- CDN resources (PDF.js, Mammoth, Google Identity Services)

## Architecture

The application follows an MVC pattern with these main classes (all in `index.html`):

- **EmailReaderModel** (~line 494): Application state management with observer pattern. States include `initializing`, `welcome`, `idle`, `loading`, `reading`, `dictating`, `confirming-*`
- **EmailReaderView** (~line 581): DOM manipulation, button visibility, status display
- **EmailUtils** (~line 866): Static utilities for email processing, HTML sanitization, attachment extraction
- **SpeechService** (~line 1114): Web Speech API wrapper for TTS and speech recognition, with sentence chunking for mobile compatibility
- **GmailService** (~line 1368): Gmail API v1 integration with OAuth
- **EmailReaderController** (~line 1674): Command routing, voice command matching (16+ commands with regex patterns), multi-step confirmation flows

## Key Implementation Details

- **Voice commands** use flexible regex matching to handle speech recognition variations (e.g., "archive", "are by", "barberry" all match the archive command)
- **iOS audio** requires special handling—audio context needs user interaction before playback
- **Speech synthesis** breaks content into sentences for mobile browser compatibility
- **HTML sanitization** removes `<script>` tags to prevent injection
- **Gmail API calls** have 100ms delays for rate limiting

## External Dependencies (CDN-loaded)

- Google Identity Services
- PDF.js 3.11.174
- Mammoth 1.6.0 (DOCX parser)

## File Structure

```
index.html    - Complete application (HTML + CSS + JavaScript, ~2600 lines)
logo.png      - Brand logo
```
