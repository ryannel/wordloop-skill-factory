# Security & Autonomous Delivery

We operate a security-first, industrialized container supply chain and a zero-trust delivery engine. The platform must automate these guarantees so developers are secure by default.

## Container Industrialization
* **Daemonless Operations**: We use **Podman** and **Buildah** for image construction instead of Docker. By eliminating the root-privileged Docker daemon, we significantly reduce the attack surface area on CI/CD build nodes.
* **Docker Hardened Images (DHI)**: All services must use Docker Hardened Images. These minimalist "distroless" base images are stripped of shells, package managers, and unnecessary libraries, delivering a full runtime in a fraction of the usual package count and preventing bad actors from executing arbitrary shell payloads.
* **Supply Chain Integrity**: Every image includes a cryptographically signed **SBOM** (Software Bill of Materials) and **SLSA Provenance**. This ensures we can explicitly verify the integrity of code from the moment it was committed to its execution in production.

## Autonomous Delivery (CI/CD)
Our delivery engine relies on observability-driven quality gates tailored for microservices, entirely removing manual approvals or static credentials.

* **Workload Identity Federation (WIF)**: We have completely eliminated static, long-lived service account keys. Our CI/CD runners (like **GitHub Actions**) authenticate to Google Cloud via short-lived OIDC tokens mapped to specific repositories and branches. There are no secrets to manually rotate, leak, or manage.
* **Google Cloud Deploy**: We use managed delivery pipelines to orchestrate automated **Canary Deployments**. 
* **Observability-Driven Development (ODD)**: We use live OTLP signals from staging and canaries to decide if code is production-ready. If a new deployment increases P99 latency by >5% or spikes error rates above threshold, an automatic rollback is immediately triggered by Cloud Deploy. No human intervention is required for rollbacks.
* **Contract Testing**: To keep integrations fast, we prioritize consumer-driven contract tests to ensure microservice compatibility at build time, rather than relying on brittle, monolithic staging environments.
