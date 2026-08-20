# Charts Tags Handling in Y-Stream Releases

## Overview

The konflux-release-data onboarding process now includes special handling for charts files during Y stream (non-EA) releases. Specifically, it adds a `vX.Y.0` tag entry to the `spec.data.mapping.defaults.tags` list.

## File Locations

The logic applies to RPA files matching these patterns:
- `*-charts-prod.yaml`
- `*-charts-stage.yaml`

Example paths:
- `data/rpa/product/rhoai/rhoai-onprem-vx-y-charts-prod.yaml`
- `data/rpa/product/rhoai/rhoai-onprem-vx-y-charts-stage.yaml`
- `data/rpa/service/rhoai/rhoai-onprem-vx-y-charts-prod.yaml`
- `data/rpa/service/rhoai/rhoai-onprem-vx-y-charts-stage.yaml`

## Behavior

### Y Stream (Non-EA) Releases

For standard Y stream releases (e.g., `3.4` → `3.5`):

**Action:** Add `vX.Y.0` to the `spec.data.mapping.defaults.tags` list

**Example:**
```yaml
# Before (source: rhoai-onprem-v3-4-charts-prod.yaml)
apiVersion: appstudio.redhat.com/v1alpha1
kind: ReleasePlanAdmission
metadata:
  name: rhoai-onprem-v3-4-charts-prod
spec:
  applications:
    - rhoai
  data:
    mapping:
      defaults:
        tags:
          - latest
          - v3.4.0
    product_version: "3.4"

# After (result: rhoai-onprem-v3-5-charts-prod.yaml)
apiVersion: appstudio.redhat.com/v1alpha1
kind: ReleasePlanAdmission
metadata:
  name: rhoai-onprem-v3-5-charts-prod
spec:
  applications:
    - rhoai
  data:
    mapping:
      defaults:
        tags:
          - latest
          - v3.4.0
          - v3.5.0        # ← NEW TAG ADDED
    product_version: "3.5"
```

### EA (Early Access) Releases

For EA releases (e.g., `3.4` → `3.5-ea.1` or `3.4-ea.1` → `3.5-ea.1`):

**Action:** No changes to the tags list

**Reasoning:** EA releases don't add version-specific tags to the defaults.

### EA → Y Stream Transitions

For transitions from EA to Y stream (e.g., `3.5-ea.1` → `3.5`):

**Action:** Add `vX.Y.0` to the `spec.data.mapping.defaults.tags` list

**Important:** The logic checks **only the target version**, not the source. This ensures that EA→Y stream transitions correctly add the tag for the Y stream release.

## Implementation Details

### Code Location
- **File:** `.claude/skills/common/scripts/rhoai_release/file_ops.py`
- **Function:** `create_rpa_files()`
- **Lines:** 219-255

### Logic Flow
1. After copying and renaming RPA files from previous to new version
2. After applying standard version replacements
3. After fixing `product_version` for EA releases
4. **New step:** For charts files in Y stream releases:
   - Parse YAML structure
   - Navigate to `spec.data.mapping.defaults.tags`
   - Append `vX.Y.0` if not already present
   - Write back as properly formatted YAML

### Code Snippet
```python
# Handle charts files: add vX.Y.0 tag to mapping.defaults.tags for Y stream (non-EA)
is_charts_file = "charts-prod.yaml" in f.name or "charts-stage.yaml" in f.name
# Check only the target version (new_version), not is_ea flag (which considers both source and target)
# This ensures EA→Y stream transitions (e.g., 3.5-ea.1 → 3.5) correctly add the tag
target_is_ea = "-ea-" in new_version_dash
if is_charts_file and not target_is_ea:
    # For Y stream releases, add vX.Y.0 entry to mapping.defaults.tags
    new_tag = f"v{new_version_dash.replace('-', '.')}.0"

    # Parse YAML to add the tag entry properly
    try:
        data = yaml.safe_load(raw)
        if data and isinstance(data, dict):
            # Navigate to spec.data.mapping.defaults.tags
            spec = data.get("spec", {})
            spec_data = spec.get("data", {})
            mapping = spec_data.get("mapping", {})
            defaults = mapping.get("defaults", {})
            tags = defaults.get("tags", [])

            # Add the new tag if it's not already present
            if new_tag not in tags:
                tags.append(new_tag)
                defaults["tags"] = tags
                mapping["defaults"] = defaults
                spec_data["mapping"] = mapping
                spec["data"] = spec_data
                data["spec"] = spec

                # Write back as YAML
                raw = yaml.dump(
                    data,
                    default_flow_style=False,
                    sort_keys=False,
                    allow_unicode=True,
                    width=1000,
                )
                logger.debug("Added tag %r to mapping.defaults.tags in %s", new_tag, f.name)
    except yaml.YAMLError as e:
        logger.warning("Could not parse YAML in %s to add tag: %s", f.name, e)
```

## Testing

A test script is provided at `test_charts_tags.py` that demonstrates:

1. **Y Stream Release (3.4 → 3.5):** Tag `v3.5.0` added to tags list ✓
2. **EA Release (3.4 → 3.5-ea.1):** No changes to tags list ✓
3. **Duplicate Prevention:** If `v3.5.0` already exists, it's not duplicated ✓
4. **Empty Tags List:** Tag is added even when tags list starts empty ✓

Run tests:
```bash
python3 test_charts_tags.py
```

Expected output: All tests pass with `✓ PASS`

## Version Transition Examples

| Transition Type | Source | Target | new_version_dash | Tag Added | Notes |
|----------------|--------|--------|------------------|-----------|-------|
| Y → Y Stream | `3.4` | `3.5` | `3-5` | `v3.5.0` | Standard Y stream |
| Y → Y Stream | `3.5` | `3.6` | `3-6` | `v3.6.0` | Standard Y stream |
| Y → EA | `3.4` | `3.5-ea.1` | `3-5-ea-1` | (none) | EA target, no tag |
| EA → EA | `3.4-ea.1` | `3.5-ea.1` | `3-5-ea-1` | (none) | EA target, no tag |
| **EA → Y** | **`3.5-ea.1`** | **`3.5`** | **`3-5`** | **`v3.5.0`** | **Critical: Y target, tag added** |
| EA → Y | `3.4-ea.2` | `3.5` | `3-5` | `v3.5.0` | Y target, tag added |

## Key Features

✅ **YAML-aware:** Properly parses and writes YAML structure  
✅ **Idempotent:** Checks if tag already exists before adding  
✅ **EA-aware:** Only affects Y stream releases, not EA  
✅ **Error handling:** Gracefully handles YAML parsing errors  
✅ **Logging:** Debug logs when tag is added

## Impact

This change affects:
- **RPA Product files:** `data/rpa/product/rhoai/*-charts-prod.yaml` and `*-charts-stage.yaml`
- **RPA Service files:** `data/rpa/service/rhoai/*-charts-prod.yaml` and `*-charts-stage.yaml`
- **Only Y stream releases:** EA releases are unaffected

Files without `charts-prod.yaml` or `charts-stage.yaml` in their name continue to use standard version replacement logic.

## Logging

The implementation includes debug logging:
- Success: `"Added tag 'vX.Y.0' to mapping.defaults.tags in <filename>"`
- Warning: `"Could not parse YAML in <filename> to add tag: <error>"`

This helps trace the transformation during onboarding runs.

## YAML Path

The tag is added to the following YAML path:
```
spec
  └─ data
      └─ mapping
          └─ defaults
              └─ tags  ← vX.Y.0 appended here
```
