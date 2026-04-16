Please read through all of the resource material listed below and apply the skill-creator skill to synthesize these resources into a comprehensive skill for Test Architecture.

This skill should guide AI agents in architecting and operating resilient testing strategies, moving away from brittle solitary unit tests and staged integration tests. It must focus on High-Fidelity Local Emulation, Risk-Based Engineering, and Observability-Driven Development.

**Scope Definition**: The scope of this skill must guide the AI in designing testing architecture according to our specific Developer Testing Handbook patterns:
* **The S/M/L Taxonomy & BDD Naming:** Structuring tests by environmental boundaries and consistent Fully Qualified Names.
* **The Four Validation Layers:** Layer 1 (Service Test), Layer 2 (Live Service Test), Layer 3 (System Test), and Layer 4 (Live System Test).
* **High-Fidelity Emulation:** Utilizing Testcontainers, LocalStack, Prism, and Ollama over generic mocks.
* **Trace-Driven Testing:** Validating OpenTelemetry (OTel) traces end-to-end as a core requirement of System Testing.
* **AI-Augmented Quality & Contracts:** Integrating Consumer-Driven Contract Testing (CDCT) and generating synthetic data.

It must align with these best practices to ensure continuous risk assurance.

To do this effectively, please follow this rigorous process:
1. **Analyze & Gap Assessment**: Conduct a deep gap analysis comparing any existing test practices against the standards in the provided testing handbook.
2. **Research & Synthesize**: Perform independent research to fill any identified gaps related to these specific architectural patterns (e.g., eBPF continuous profiling, Testcontainers orchestration).
3. **Modular Restructure**: Synthesize the findings and fully decompose the guidance into a modular structure following the progressive disclosure methodology documented in the skill-creator. Break the reference library down into strict single-concern modules (e.g., "emulation-strategy", "trace-validation", "cdct-implementation").

Put the new skill and all its child files/folders in a `/skills/test-architect` folder.

Please be extremely thorough, working through each and every relevant topic in the handbook. Ensure full structural parity across testing levels, frameworks, methodologies, and toolchains within the output skill. The output must strictly avoid documented antipatterns like solitary unit testing, integrated testing gaps, and reliance on shared staging environments.

# Resources

## Applied Skills
* .agents/skills/skill-creator

## Reference Documentation Folder
Be sure to read every single file in these folders:
* /resources/test-architect/*

# What not to do:
* The skill should not reference the resources files directly but apply its contents. 
* The skill should not reference the year or the year of any trends or resources. 
* Don't read or reference any other documents in the repo besides the ones specified in the resources section.
