# Security Policy

## Reporting a Vulnerability

If you find a security vulnerability in Perspective Cuts, please report it responsibly. Do not open a public issue.

Email me directly or use GitHub's private vulnerability reporting feature on this repository.

## What Counts as a Security Issue

- The compiler generating shortcuts that execute unintended actions
- The `--install` flag modifying shortcuts other than the one being installed
- The `discover` command exposing sensitive data from the ToolKit database
- Any code injection through `.perspective` file parsing

## What Does Not Count

- Apple Shortcuts themselves requesting permissions (that is Apple's domain)
- Third party app actions requiring authentication (that is the app's responsibility)
- The `shortcuts sign` command behavior (that is Apple's tool)

## Scope

Perspective Cuts is a compiler. It generates plist files and signs them with Apple's tool. It does not have network access, does not store credentials, and does not run shortcuts itself. The security surface is the compiler input (`.perspective` files) and the database access (`--install` and `discover` commands).
