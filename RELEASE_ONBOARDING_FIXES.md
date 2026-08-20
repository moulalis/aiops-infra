# RHOAI Release Onboarding Fixes

## Summary

Two critical fixes have been implemented for the RHOAI Y-stream release onboarding automation:

1. **RHOAI-Build-Config**: Fixed `#!scmRevision` directive handling
2. **konflux-release-data**: Fixed charts RPA tags to include all three required tags

## Issue 1: RHOAI-Build-Config `config/trustyai-pig-build-config.yaml`

### Problem

When onboarding a new release (e.g., `rhoai-3.5-ea.2 → rhoai-3.5`), the `config/trustyai-pig-build-config.yaml` file had:
- ✅ `#!productVersion=3.5.0` (correct)
- ❌ `#!scmRevision=rhoai-3.5-ea.2` (incorrect - should be `rhoai-3.5`)

### Root Cause

The `update_config_trustyai_build()` function only updated `#!productVersion` but didn't update `#!scmRevision`.

### Fix

Updated `rbc_release.py` to handle **both** directives:

```python
def update_config_trustyai_build(new_major, new_minor, new_version_with_prefix):
    """
    Update config/trustyai-pig-build-config.yaml:
    - #!productVersion= uses x.y.0 format only (no EA suffix)
    - #!scmRevision= uses the full target version (with EA suffix if applicable)
    """
    # ... implementation ...
    
    # Replace #!productVersion= line
    pattern_product = r"^#!productVersion=.*$"
    replacement_product = f"#!productVersion={target_product_version}"
    content = re.sub(pattern_product, replacement_product, content, flags=re.MULTILINE)

    # Replace #!scmRevision= line
    pattern_scm = r"^#!scmRevision=.*$"
    replacement_scm = f"#!scmRevision={target_scm_revision}"
    content = re.sub(pattern_scm, replacement_scm, content, flags=re.MULTILINE)
```

### Expected Results

| Transition | #!productVersion | #!scmRevision |
|------------|------------------|---------------|
| 3.4 → 3.5 | 3.5.0 | rhoai-3.5 |
| 3.4 → 3.5-ea.1 | 3.5.0 | rhoai-3.5-ea.1 |
| 3.5-ea.1 → 3.5 | 3.5.0 | rhoai-3.5 |
| 3.5-ea.1 → 3.5-ea.2 | 3.5.0 | rhoai-3.5-ea.2 |

---

## Issue 2: konflux-release-data Charts RPA Files Tags

### Problem

When onboarding a Y-stream release (e.g., `rhoai-3.5-ea.2 → rhoai-3.5`), the charts RPA files should have gotten **three** tags:
1. `v3.5`
2. `v3.5.0-{{ release_timestamp }}`
3. `v3.5.0`

But only **one** tag was being added:
- ❌ Only `v3.5.0` was added

### Root Cause

The `create_rpa_files()` function in `file_ops.py` only added a single tag (`vX.Y.0`) instead of all three required tags.

### Fix

Updated `file_ops.py` to add **all three** tags for Y-stream releases:

```python
if is_charts_file and not target_is_ea:
    # For Y stream releases, add three tags to mapping.defaults.tags:
    # 1. vX.Y (e.g., v3.5)
    # 2. vX.Y.0-{{ release_timestamp }} (template placeholder)
    # 3. vX.Y.0 (e.g., v3.5.0)
    version_base = new_version_dash.replace('-', '.')  # e.g., "3.5"
    new_tags = [
        f"v{version_base}",
        f"v{version_base}.0-{{{{ release_timestamp }}}}",
        f"v{version_base}.0"
    ]
    
    # Add each new tag if not already present
    for tag in new_tags:
        if tag not in tags:
            tags.append(tag)
```

### Expected Results

**Y-stream release** (3.5-ea.2 → 3.5):
```yaml
spec:
  data:
    mapping:
      defaults:
        tags:
          - v3.5
          - v3.5.0-{{ release_timestamp }}
          - v3.5.0
```

**EA release** (3.4 → 3.5-ea.1):
- No tags added (EA releases don't get chart tags)

---

## Files Modified

### Core Implementation
1. `.claude/skills/common/scripts/rhoai_release/rbc_release.py`
   - Updated `update_config_trustyai_build()` function signature
   - Added `#!scmRevision` handling
   - Updated function call to pass `logical_latest` parameter

2. `.claude/skills/common/scripts/rhoai_release/file_ops.py`
   - Updated charts tags logic to add three tags instead of one
   - Added detailed comments explaining the three-tag structure

### Tests
3. `test_config_productversion.py` - Updated to test both directives
4. `test_charts_tags_updated.py` - New test for three-tag functionality

### Documentation
5. `CONFIG_PRODUCTVERSION_HANDLING.md` - Updated with both fixes
6. `RELEASE_ONBOARDING_FIXES.md` - This summary document

---

## Testing

### Test: Config File Handling
```bash
python test_config_productversion.py
```

All tests pass ✅:
- Y-stream to Y-stream
- Y-stream to EA
- EA to Y-stream
- EA to EA
- Content preservation

### Test: Charts Tags
```bash
python test_charts_tags_updated.py
```

All tests pass ✅:
- Y-stream gets three tags
- EA gets no tags
- Decision based on target version only

---

## Impact

These fixes ensure that:

1. **Build system correctness**: `#!scmRevision` points to the correct source branch
2. **Container image promotion**: All three required tags are present for Y-stream releases
3. **Consistency**: Both EA and Y-stream releases follow the same versioning logic

---

## Next Steps

When you re-run the Y-stream onboarding for the next release, these fixes will automatically be applied:

1. **RBC Release step**: `config/trustyai-pig-build-config.yaml` will have both directives updated correctly
2. **Konflux step**: Charts RPA files will include all three tags

To verify the fixes were applied:

```bash
# Check RHOAI-Build-Config
curl -s https://raw.githubusercontent.com/red-hat-data-services/RHOAI-Build-Config/<branch>/config/trustyai-pig-build-config.yaml | head -5

# Check konflux-release-data charts file
# Look for spec.data.mapping.defaults.tags in the charts-prod.yaml file
```
