# Observability & GreenOps

In the modern enterprise, observability is a fundamental development requirement, not a monitoring afterthought. We build platforms that make logging and metrics "Shift Down" out of the developer's way.

## Trace-First Observability
We prioritize distributed tracing over disparate logs and metrics to build a complete picture of the user journey.

* **Unified OTLP Ingestion**: We utilize the Google Cloud `telemetry.googleapis.com` API as our unified ingestion point. We adopt the **OpenTelemetry Protocol (OTLP)** natively. By instrumenting code with OTLP, we can embed up to 1,024 rich attributes per span, moving past simple metrics and text logs into deep **Analytical Debugging**.
* **Managed Collectors**: We implement **OpenTelemetry Collectors** on the cluster and node level to manage cost and ensure data privacy. The collectors perform probabilistic sampling (e.g., keeping 100% of errors but 1% of success traces) and mask sensitive PII before data leaves the application boundary.

## GreenOps (Sustainability)
Platform engineers are directly responsible for the carbon footprint of their compute systems.

* **Carbon Dashboards**: We integrate real-time carbon-intensity metrics directly into our core infrastructure monitoring dashboards to visualize the relationship between compute waste and emissions.
* **Carbon-Aware Scheduling**: We implement Carbon-Aware Scheduling for batch jobs, cron jobs, and asynchronous workloads. Non-critical jobs are automatically geographically shifted to regions (like Sweden's SE3/SE4 zones) or temporally shifted to times of day when the power grid is powered by 100% renewable energy.
