Please read through all of the resource material listed below and apply the skill-creator skill to synthesize these resources into a comprehensive skill for PostgreSQL schema design, data modeling, and query engineering.

This skill should guide AI agents in designing PostgreSQL schemas that are correct by construction — enforcing invariants at the database layer, selecting the right column types and indexes for the access pattern, writing safe zero-downtime migrations, and diagnosing query performance problems.

**Scope Definition**: The scope of this skill covers the full lifecycle of PostgreSQL schema work: entity modeling, column type selection, constraint design, index strategy, migration authoring, query optimization, Row-Level Security, JSONB patterns, and vector search with pgvector. It must be practical and opinionated — the agent should produce concrete SQL DDL, migration files, and `EXPLAIN ANALYZE` guidance, not generic advice.

It must be modern, aligning with best practices and the latest trend-leading patterns for production PostgreSQL deployments.

To do this effectively, please follow this rigorous process:
1. **Analyze & Gap Assessment**: Conduct a deep gap analysis comparing the existing reference skills and folder documentation against current industry standards and trend leading practices for PostgreSQL database engineering in 2026.
2. **Research & Synthesize**: Perform independent research to fill any identified gaps and deeply integrate useful patterns and practices.
3. **Modular Restructure**: Synthesize the findings and fully decompose the guidance into a modular structure following the progressive disclosure methodology documented in the skill-creator. Break the reference library down into strict single-concern modules.

Put the new skill and all its child files/folders in a `/tools/skill-factory/skills/postgres-designer` folder.

Please be extremely thorough, working through each and every relevant topic, ensuring full structural coverage across schema design, type selection, index strategy, constraints, migrations, query performance, security, and JSONB patterns, giving each the space and attention in the output skill and resource files that it deserves.

# Resources

## Applied Skills
* .agents/skills/skill-creator

## Reference Skills:
Be sure to read the the entire skill and all its resources, encorporating its information into the new skill.
* /tools/skill-factory/.agents/skills/postgresql-table-design

## Reference Documentation Folder
Be sure to read every single file in these folders:
* /tools/skill-factory/skill-blueprints/postgres-designer/resources/*

# What not to do:
* The skill should not reference the resource files directly but apply their contents.
* The skill should not reference the year or the year of any trends or resources.
* Don't read or reference any other documents in the repo besides the ones specified in the resources section.
