# Implementation Summary: Charts Tags for Y Stream Releases

## Problem Statement

For Y stream (non-EA) releases in konflux-release-data onboarding, we need to add a `vX.Y.0` tag to the `spec.data.mapping.defaults.tags` list in charts files (`*-charts-prod.yaml` and `*-charts-stage.yaml`).

## Critical Edge Case Discovered

**EA → Y Stream Transition** (e.g., `rhoai-3.5-ea.1` → `rhoai-3.5`)

The initial implementation had a bug where the `is_ea` flag in `konflux_onboard.py` is calculated as:
```python
is_ea = is_ea_version(previous_version) or is_ea_version(new_version)
```

This means for EA → Y stream transitions:
- Source: `3.5-ea.1` (EA) → `is_ea_version(previous) = True`
- Target: `3.5` (Y stream) → `is_ea_version(new) = False`
- Result: `is_ea = True` (because previous is EA)

Using `if not is_ea` would **incorrectly skip** adding the tag for Y stream targets.

## Solution

**Check only the target version**, not the `is_ea` flag:

```python
target_is_ea = "-ea-" in new_version_dash
if is_charts_file and not target_is_ea:
    # Add vX.Y.0 tag
```

This ensures the decision is based solely on whether the **target** is a Y stream release, regardless of the source version type.

## Implementation

### Modified File
`.claude/skills/common/scripts/rhoai_release/file_ops.py`

### Changes (lines 219-255)
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

## Test Coverage

### All Transition Scenarios Tested

| Transition | Source | Target | Expected Behavior | Status |
|------------|--------|--------|-------------------|--------|
| Y → Y | `3.4` | `3.5` | Add `v3.5.0` tag | ✓ PASS |
| Y → EA | `3.4` | `3.5-ea.1` | NO tag added | ✓ PASS |
| EA → EA | `3.4-ea.1` | `3.5-ea.1` | NO tag added | ✓ PASS |
| **EA → Y** | **`3.5-ea.1`** | **`3.5`** | **Add `v3.5.0` tag** | **✓ PASS** |
| EA → Y | `3.4-ea.2` | `3.5` | Add `v3.5.0` tag | ✓ PASS |
| Y → Y | `3.5` | `3.6` | Add `v3.6.0` tag | ✓ PASS |

**All 6 test scenarios pass.**

### Test Scripts

1. **`test_charts_tags.py`** - Basic functionality tests (4 tests)
2. **`test_ea_to_y_stream.py`** - Demonstrates the EA→Y problem and solution
3. **`test_all_transitions.py`** - Comprehensive test of all 6 transition types

Run: `python3 test_all_transitions.py`

## Files Affected

Charts files in RPA directories during Y stream releases:
- `data/rpa/product/rhoai/*-charts-prod.yaml`
- `data/rpa/product/rhoai/*-charts-stage.yaml`
- `data/rpa/service/rhoai/*-charts-prod.yaml`
- `data/rpa/service/rhoai/*-charts-stage.yaml`

## Example Transformation

### Before (source: v3.4-charts-prod.yaml)
```yaml
spec:
  data:
    mapping:
      defaults:
        tags:
          - latest
          - v3.4.0
    product_version: "3.4"
```

### After (target: v3.5-charts-prod.yaml for Y stream)
```yaml
spec:
  data:
    mapping:
      defaults:
        tags:
          - latest
          - v3.4.0
          - v3.5.0    # ← Added
    product_version: "3.5"
```

### After (target: v3.5-ea-1-charts-prod.yaml for EA)
```yaml
spec:
  data:
    mapping:
      defaults:
        tags:
          - latest
          - v3.4.0    # ← No new tag added
    product_version: "3.5"
```

## Key Features

✅ **Target-based decision** - Checks only target version, not source  
✅ **EA → Y stream support** - Correctly handles EA to Y stream transitions  
✅ **YAML-aware** - Properly parses and writes YAML structure  
✅ **Idempotent** - Prevents duplicate tags  
✅ **Error handling** - Gracefully handles YAML parsing errors  
✅ **Comprehensive logging** - Debug logs for traceability  

## Documentation

- **`CHARTS_TAGS_HANDLING.md`** - Complete technical documentation
- **`IMPLEMENTATION_SUMMARY.md`** - This file (high-level summary)

## Answer to Original Question

> "What if source release is rhoai-3.5-ea.1 and target release is rhoai-3.5? Does it work perfectly?"

**YES**, it now works perfectly! The fixed implementation:

1. ✅ Detects that target `3.5` is a Y stream release (not EA)
2. ✅ Adds `v3.5.0` to the `mapping.defaults.tags` list
3. ✅ Ignores the fact that source was `3.5-ea.1`
4. ✅ Passes comprehensive tests including this exact scenario

The key insight: **Always check the target version type, not the source or a combined flag.**
