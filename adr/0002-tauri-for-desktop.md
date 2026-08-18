# 0002. Tauri for the desktop application

Date: 2026-08-18
Status: Accepted

## Context

Scalar needs a desktop application for macOS, Windows and Linux with a global shortcut for Command, tray presence, native notifications, launch at startup, secure credential storage and background sync. The interface is already built in React for the web.

## Decision

The desktop application is built with Tauri: a Rust native shell hosting the React and TypeScript interface built with Vite, sharing `@scalar/ui` and `@scalar/sdk` with the web app. Electron is not used unless a platform limitation makes Tauri unsuitable, and that exception would need its own record.

## Consequences

- Small binaries and low memory use compared with Electron.
- Native capabilities are written in Rust behind Tauri commands; contributors touching those need Rust.
- Web view differences between platforms must be tested (WebKit on macOS, WebView2 on Windows, WebKitGTK on Linux).

## Alternatives considered

- Electron: mature, uniform Chromium, but heavy and at odds with a lightweight always running companion.
- Native per platform (Swift, WinUI, GTK): three codebases for the same interface.
- Progressive web app only: no global shortcut, tray or reliable background sync.
