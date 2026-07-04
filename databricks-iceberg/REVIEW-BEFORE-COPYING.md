# Review Before Copying: databricks-iceberg

## Files Changed

1. `SKILL.md`
2. `references/1-managed-iceberg-tables.md`
3. `references/5-external-engine-interop.md`

**Unchanged** (copied as-is):
- `references/2-uniform-and-compatibility.md`
- `references/3-iceberg-rest-catalog.md`
- `references/4-snowflake-interop.md`

---

## What Changed and Why

### SKILL.md

| Change | Why |
|--------|-----|
| Version `0.1.0` → `0.2.0` | This update |
| Iceberg v3 description: "Beta, DBR 17.3+" → "GA, DBR 18.0+" | Iceberg v3 is now GA |
| Added "Geospatial types require DBR 18.2+" to Iceberg v3 description | New requirement |
| Common Issues: version mismatch row: `1.9.0+` → `1.9.2+` | Library version update |
| Added "Small files accumulating" common issue pointing to Predictive Optimization | New PO requirement |
| Resources section: Iceberg v3 link description changed from "Beta" to "GA" | Status update |

### references/1-managed-iceberg-tables.md

| Change | Why |
|--------|-----|
| Requirements header: `DBR 17.3+ (Managed Iceberg v3 Beta)` → `DBR 18.0+ (Managed Iceberg v3 GA)` | Iceberg v3 GA |
| Iceberg v3 section heading: "Iceberg v3 (Beta)" → "Iceberg v3 (GA)" | Status update |
| Iceberg v3 "Requires" line: `DBR 17.3+` → `DBR 18.0+` | DBR requirement update |
| Added "Geospatial Types" row to v3 features table: "Requires DBR 18.2+" | New requirement |
| Iceberg v3 "Beta status" note → "GA status" note: suitable for production | Status update |
| External engine compatibility: `1.9.0+` → `1.9.2+` | Library version update |
| **Added entire "Predictive Optimization (Required for Automatic Maintenance)" section** | PO is not auto-enabled; without it tables accumulate small files |
| Added `ALTER TABLE ... SET TBLPROPERTIES ('delta.enablePredictiveOptimization' = 'auto')` | Canonical enablement command |
| Added catalog/schema-level PO enablement examples | PO best practices |
| **Added "Deletion Vectors on Iceberg v3" subsection** | DVs are on by default in v3; `iceberg.enableDeletionVectors` is the correct property |
| Added: DVs enabled by default on v3, controlling property is `'iceberg.enableDeletionVectors'` | Correct property name |
| Added: disable DVs with `'iceberg.enableDeletionVectors' = 'false'` when CLUSTER BY is needed | Use case for disabling |
| Updated `CLUSTER BY` on v3 example to use correct DV property | Accuracy |

### references/5-external-engine-interop.md

| Change | Why |
|--------|-----|
| OSS Spark `ICEBERG_VER`: `1.7.1` → `1.9.2` | Library version update: minimum for v3 support |
| Updated comment next to version: "minimum for Iceberg v3 support" | Clarity |
| Troubleshooting "v3 table incompatibility": `1.9.0+` → `1.9.2+` | Library version update |

---

## Copy Commands

```bash
# Copy all updated files to the installed skill directory
SKILL_DIR="$HOME/.databricks/aitools/skills/databricks-iceberg"
UPDATE_DIR="$HOME/skill-updates/databricks-iceberg"

cp "$UPDATE_DIR/SKILL.md"                                          "$SKILL_DIR/SKILL.md"
cp "$UPDATE_DIR/references/1-managed-iceberg-tables.md"            "$SKILL_DIR/references/1-managed-iceberg-tables.md"
cp "$UPDATE_DIR/references/5-external-engine-interop.md"           "$SKILL_DIR/references/5-external-engine-interop.md"

# Note: references/2, 3, 4 are unchanged — no copy needed unless verifying
# cp "$UPDATE_DIR/references/2-uniform-and-compatibility.md"       "$SKILL_DIR/references/2-uniform-and-compatibility.md"
# cp "$UPDATE_DIR/references/3-iceberg-rest-catalog.md"            "$SKILL_DIR/references/3-iceberg-rest-catalog.md"
# cp "$UPDATE_DIR/references/4-snowflake-interop.md"               "$SKILL_DIR/references/4-snowflake-interop.md"

echo "Done. Verify with: head -5 $SKILL_DIR/SKILL.md"
```
