# Intelexta Schemas & Protocol
### An Open Standard for Verifiable AI Workflows
A project by [Gauge Freedom, Inc.](https://gaugefreedom.com)

Our motto: **"Proof, not vibes."**

This repository contains the open-source schemas for the **Content-Addressable Receipt (CAR)**, the core of Intelexta's cryptographic "trust layer" for the AI economy.

## The Problem
The AI economy is a "black box." Workflows are irreproducible, outputs are unverifiable, and costs are opaque. We cannot build a trustworthy future on this foundation.

## The Solution: The "CAR" Protocol
The **Content-Addressable Receipt (CAR)** is a portable, tamper-evident, and cryptographically verifiable "proof certificate" for any AI-driven workflow.

By open-sourcing this standard, we aim to provide a common protocol for developers, researchers, and enterprises to create and verify AI workflows with certainty.

### Key Features of the CAR Protocol:
* **Cryptographic Proof:** Uses hash chains and Ed25519 digital signatures to prove workflow integrity.
* **Tamper-Evident:** Any modification to prompts, models, or outputs is immediately detected.
* **Accountable:** Tracks resource governance, including API cost, token usage, and environmental impact (carbon footprint).
* **Verifiable:** Can be verified by our lightweight, standalone CLI tool, `intelexta-verify` (coming soon).

### Current Schema
You can review the current stable version of the protocol here:
* `[car-v0.2.schema.json](car-v0.2.schema.json)`

## What is Intelexta?
**Intelexta** is our commercial, local-first, Rust-based IDE that uses this protocol to empower AI creators. Think of it as the **"GitHub for Prompts"**—a professional workspace that brings rigor, mastery, and collaboration to the AI R&D process.

By building on an open standard, we are ensuring our users are never locked in and that the proofs they generate are verifiable by anyone, anywhere.

---
Learn more about our mission and products at **[gaugefreedom.com](https://gaugefreedom.com)**.
