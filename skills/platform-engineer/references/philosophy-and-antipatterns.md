# Philosophy & Antipatterns

## The Dual Mandate
The goal of the modern Internal Developer Platform (IDP) is the Dual Mandate:
1. Provide an industrialized software supply chain.
2. Turbocharge developer productivity.

Success is measured via the **SPACE Framework** and the **Platform Efficiency Ratio (PER)**:
`PER = (Feature Cycles Delivered) / (Infrastructure Toil Hours + Mean Time to Recovery)`

## Core Principles

### Platform-as-a-Product
Treat the platform as a product with its own lifecycle and customers (the developers). 

### Intrinsic Pull
We do not mandate adoption through policy alone. We build tools that are so significantly easier and faster to use than the alternative that teams choose them voluntarily.

### Shifting Down
We replace the outdated "Shift Left" (which burdened developers with operations tasks) with **"Shifting Down"**. We embed security, compliance, and observability directly into the platform substrate, ensuring services are secure and observable by default without increasing developer cognitive load.

### The Golden Path
We provide highly opinionated, automated workflows that allow product teams to navigate the entire lifecycle—from "Hello World" to "Production Carbon-Aware"—independently.

---

## Antipatterns: What We Don't Do

To maintain Elite delivery performance, we explicitly avoid the following:

* **The "Portal Trap"**: Building a developer portal (like Backstage) without the underlying orchestration logic. A portal is a catalog; a platform is the automation that actually eliminates toil.
* **Ticket-Ops**: We do not allow manual tickets for infrastructure or access. If it isn't a self-service API, it isn't part of the platform.
* **Shift-Left Exhaustion**: We do not dump infrastructure complexity onto developers. We "Shift Down" so they can focus on shipping business logic.
* **Long-Lived "Dev" Clusters**: We avoid static environments that naturally drift and rot. We use ephemeral PR environments that match production exactly.
* **Static Secret Management**: We do not store long-lived service account keys in GitHub Secrets or vault systems. We rely exclusively on Workload Identity Federation (WIF).
