---
name: databricks-dabs
description: 'Create, configure, validate, deploy, run, and manage Declarative Automation Bundles (DABs, formerly Databricks Asset Bundles). Use when working with Databricks resources via DABs including dashboards, jobs, pipelines, alerts, volumes, and apps.'
compatibility: Requires databricks CLI (>= v1.0.0)
metadata:
  version: "0.2.0"
---

# Declarative Automation Bundles (DABs)

Use this skill for any bundle-related request including creating, configuring, validating, deploying, running, and managing Databricks resources through DABs.

> **CLI floor:** DABs guidance in this skill assumes Databricks CLI **v1.0.0 or newer**. Verify your version with `databricks --version` and upgrade if you are on an earlier release.

## Reference Documentation

The following reference files provide detailed guidance for specific bundle tasks:

- **[Bundle Structure](references/bundle-structure.md)** - Bundle structure, databricks.yml configuration, resource definitions, path resolution, variables, and multi-environment targets
- **[SDP Pipelines](references/sdp-pipelines.md)** - Spark Declarative Pipeline configurations for DABs
- **[SQL Alerts](references/alerts.md)** - SQL Alert schemas and configuration (critical - API differs from other resources)
- **[Deploy and Run](references/deploy-and-run.md)** - Validation, deployment, running resources, monitoring logs, and troubleshooting common issues
- **[Resource Permissions](references/resource-permissions.md)** - Permission levels and access control for bundle resources, per-resource-type levels, grants vs permissions

## When to Use This Skill

Load this skill for any request involving:

- Creating new bundle projects or resources
- Configuring databricks.yml or resource YAML files
- Setting up multi-environment deployments (dev/prod targets)
- Deploying or running bundle resources
- Managing permissions for bundle resources
- Troubleshooting bundle validation or deployment errors
- Working with specific resource types (dashboards, jobs, pipelines, alerts, volumes, apps)

## General Guidelines

1. **Always validate after configuration changes** - Run `bundle validate --target <target>` after any change, then run `bundle summary --target <target>` to confirm the resolved configuration matches your intent
2. **Use reference documentation** - Consult the appropriate reference file for detailed patterns and examples
3. **Follow naming conventions** - Resource files should use `<name>.<resource_type>.yml` format
4. **Path resolution is critical** - Paths differ based on file location (see Bundle Structure reference)
5. **Preserve existing structure** - Keep user comments and structure when editing YAML files
6. **Use variables** - Parameterize catalog, schema, and warehouse for multi-environment support
7. **Exclude generated dashboard artifacts from sync** - Add `*.lvdash.json` to the bundle `sync.exclude` list so serialized AI/BI dashboard files are not re-uploaded on every deploy (see note below)

## Excluding `.lvdash.json` from `bundle sync`

Serialized AI/BI (Lakeview) dashboard files use the `*.lvdash.json` extension. These are generated artifacts and can cause noisy or conflicting uploads during `bundle sync`/`bundle deploy`. Exclude them explicitly:

```yaml
# databricks.yml
sync:
  exclude:
    - '*.lvdash.json'
```

The dashboard resource still references its `.lvdash.json` via `file_path`; the exclusion only prevents the raw file from being copied by the general sync step.

## Documentation

- [Declarative Automation Bundles](https://docs.databricks.com/dev-tools/bundles/)
- [Bundle Examples Repository](https://github.com/databricks/bundle-examples) - official end-to-end example bundles (jobs, pipelines, dashboards, apps, and more); use as working references for patterns not covered by the reference files above
