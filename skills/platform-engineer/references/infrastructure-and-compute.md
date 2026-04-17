# Infrastructure, Compute, & Networking

## Intent-Based Control Planes
We treat our infrastructure as a versioned, testable, and self-healing software product. Rather than manually scripting resource creation imperatively, we rely on control loops.

* **Continuous Reconciliation**: While we utilize Terraform for initial bootstrapping, our trend-leading standard is the use of **Crossplane** and **Google Cloud Config Connector**. We manage infrastructure as living resources; if a configuration drifts out of band, the control plane automatically "heals" it back to the desired state.
* **Intent-Based Abstractions**: Developers define *intent* (e.g., "I need a scalable API with a 1GB Firestore database") using simplified, domain-specific YAML. The platform translates this intent into the hundreds of lines of HCL or YAML required for VPCs, IAM, and compute resources behind the scenes.
* **Policy-as-Code (PaC)**: We utilize **Open Policy Agent (OPA)** to evaluate every infrastructure plan. Non-compliant resources (e.g., unencrypted buckets or oversized instances) are automatically blocked before they are provisioned.

## Compute: Serverless & Thin Clients
Our architecture is optimized for "thin client, smart backend" designs where logic is decoupled from state.

* **Cloud Run First**: We utilize **Google Cloud Run** as our primary compute environment. It provides a managed, autoscaling container environment that scales to zero, significantly reducing waste.
* **The Logic Distribution Rule**: Complex, long-running business logic resides in **Cloud Run**, while triggers, webhooks, and simple event-driven glue workflows reside in **Cloud Run Functions**.

## Mesh-less Networking
* **Google Cloud Service Connect**: We utilize Service Connect for internal service communication. This provides the security, load balancing, and observability of a service mesh without the "Sidecar Tax" or operational overhead of managing a heavyweight data plane like Istio.

## Planet-Scale Data
* **Firestore**: Used for real-time NoSQL state. Always ensure we prevent "hotspotting" by using scatter algorithms (like hashing or UUIDs) for document IDs rather than monotonic sequential keys.
* **BigQuery**: Analytical workloads are continuously streamed to BigQuery for high-performance business intelligence, keeping our transactional stores lightweight.
