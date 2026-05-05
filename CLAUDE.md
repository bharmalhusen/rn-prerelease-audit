# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A Claude Code skills plugin that performs a deep pre-release audit of React Native apps. It is structured as a standard Claude plugin with a skill definition, plugin manifest, and marketplace metadata.

## Plugin structure

```
.claude-plugin/
  plugin.json          — plugin manifest (name, version, skills pointer)
  marketplace.json     — marketplace listing metadata
skills/
  rn-prerelease-audit/
    SKILL.md           — the audit skill (invokable as /rn-prerelease-audit)
```

## Current skills

### `skills/rn-prerelease-audit/SKILL.md` — RN Pre-Release Audit

A Claude Code skill that audits a React Native app before App Store or Play Store submission. It instructs the agent to:

1. **Read project files** — `AndroidManifest.xml`, `build.gradle`, `Info.plist`, `app.json`, `Podfile`, `package.json`, and source files for permission and analytics usage.
2. **Run 30+ structured checks** across 20+ categories: ATT, Sign in with Apple, auth session handling, payments/billing, permissions, icons, splash screen, iOS privacy descriptions, network security, hardcoded secrets, crash monitoring, New Architecture flags, network resilience, deep links, deprecated dependencies, account deletion, privacy policy, Target SDK, back-press handling, Privacy Manifests, ProGuard, export compliance, Hermes, bundle size, UI safety, Android activity config, version consistency, and screen orientation.
3. **Output a formatted report** with severity tiers (🔴 errors, 🟡 warnings, 🔵 info, ✅ passed) and a prioritized next-steps list.

## How to install and use

Install as a Claude Code plugin, then run inside any React Native project:

```
/rn-prerelease-audit
```
