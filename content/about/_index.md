---
date: 2022-01-18T20:24:13-07:00
title: About Pkgcraft
meta: false
---

Pkgcraft is a highly experimental, rust-based, tooling ecosystem for Gentoo.

# Project goals

## Short-term

- Wrap bash in a rust-based library allowing process reuse and optional static
  binary build support.

- Write all native bash functionality that would otherwise be required using
  rust-based builtins and shell functionality.

## Long-term

- Develop a threaded, semi-modular resolver and constraint framework that
  provides a more flexible approach to dependency resolution.

- Support various frontends for package building. Currently a simple
  command-line tool is planned but an interactive terminal interface and web
  frontend would be great.

- Provide bindings that allow users to hook into pkgcraft-functionality from
  their language of choice. Currently basic C and Python bindings exist
  wrapping a minimal set of package atom features.

- Support building custom targets, e.g. containers, images, tarballs, etc.

- Develop a new sandboxing framework for segregating package builds from the
  system.

## Dreams

- Replace bash with something threadable, extensible, and modern.
