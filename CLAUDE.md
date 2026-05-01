# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A collection of Claude agent prompt definitions — structured markdown files that describe how an agent should behave, what files to read, what checks to run, and how to format its output. The primary file is `audit.md`.

## Current agents

### `audit.md` — RN Pre-Release Audit Agent

A system prompt for a React Native app pre-release audit agent. It instructs the agent to:

1. **Read project files** — `AndroidManifest.xml`, `build.gradle`, `Info.plist`, `app.json`, `Podfile`, `package.json`, and source files for permission usage.
2. **Run structured checks** across 10 categories: permissions, icons/assets, splash screen, iOS privacy descriptions, network security, hardcoded secrets, New Architecture flags, deep links, deprecated dependencies, Android activity config, version consistency, and screen orientation.
3. **Output a formatted report** with severity tiers (🔴 errors, 🟡 warnings, 🔵 info, ✅ passed) and a prioritized next-steps list.

## How to use an agent prompt

Paste the contents of a `.md` file as the system prompt (or first user message) when starting a Claude conversation, then point it at the target React Native project.
