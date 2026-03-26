# Copilot

A collection of AI workflows and tools for GitHub Copilot.

## Structure

This repository is organized by dedicated workflows:

- **[tdd-agents/](tdd-agents/)** — Test-driven development (TDD) multi-agent workflow
  - Three specialized agents that collaborate to implement features following TDD principles
  - Shared scratchpad communication system for agent handoffs
  - See [tdd-agents/README.md](tdd-agents/README.md) for complete documentation

## Installation

The repository contains Markdown-based agent and skill definitions; there is no
build step. To use or customize the agents and skills, copy/symlink the
specific files into your customization directory.
