# Config Build File Handling

## Overview

The `config/trustyai-pig-build-config.yaml` file in the RHOAI-Build-Config repository requires special handling for two directives during release branch onboarding:
1. `#!productVersion=` — Always uses `x.y.0` format
2. `#!scmRevision=` — Uses the target version (with EA suffix if applicable)

## Requirements

### #!productVersion=

The `#!productVersion=` field must **always** use the `x.y.0` format, regardless of whether the release is EA or Y-stream:

- ✅ Y-stream release (3.4 → 3.5): `#!productVersion=3.5.0`
- ✅ EA release (3.4 → 3.5-ea.1): `#!productVersion=3.5.0` (NOT `3.5-ea.1.0`)
- ✅ EA to Y-stream (3.5-ea.1 → 3.5): `#!productVersion=3.5.0`
- ✅ EA to EA (3.5-ea.1 → 3.5-ea.2): `#!productVersion=3.5.0`

### #!scmRevision=

The `#!scmRevision=` field must match the **target version** exactly (including EA suffix if applicable):

- ✅ Y-stream release (3.4 → 3.5): `#!scmRevision=rhoai-3.5`
- ✅ EA release (3.4 → 3.5-ea.1): `#!scmRevision=rhoai-3.5-ea.1`
- ✅ EA to Y-stream (3.5-ea.1 → 3.5): `#!scmRevision=rhoai-3.5`
- ✅ EA to EA (3.5-ea.1 → 3.5-ea.2): `#!scmRevision=rhoai-3.5-ea.2`

## Implementation

### Function: `update_config_trustyai_build()`

Location: `.claude/skills/common/scripts/rhoai_release/rbc_release.py`

```python
def update_config_trustyai_build(new_major, new_minor):
    """
    Update config/trustyai-pig-build-config.yaml to ensure #!productVersion= uses x.y.0 format only.
    For both Y-stream and EA releases, the productVersion should be x.y.0 (e.g., 3.5.0), never with EA suffix.
    """
```

### Integration Point

The function is called in `main()` after the standard version replacements:

```python
tekton_final = update_tekton_files(mapping, tekton_paths)
update_bundle_patch(mapping)
update_csv_patch(mapping, skip_for_ea_train=skip_csv_same_train_ea)
config_updated = update_config_trustyai_build(new_major, new_minor)  # <-- Added here
```

The updated config file is automatically staged for commit if changes were made.

## Example File

### Before (on rhoai-3.4 branch):
```yaml
#!productVersion=3.4.0
#!scmRevision=rhoai-3.4

product:
  name: Red Hat OpenShift Data Science
  version: {{productVersion}}
```

### After Y-stream onboarding (3.4 → 3.5):
```yaml
#!productVersion=3.5.0
#!scmRevision=rhoai-3.5

product:
  name: Red Hat OpenShift Data Science
  version: {{productVersion}}
```

### After EA onboarding (3.4 → 3.5-ea.1):
```yaml
#!productVersion=3.5.0
#!scmRevision=rhoai-3.5-ea.1

product:
  name: Red Hat OpenShift Data Science
  version: {{productVersion}}
```

**Note:** `#!scmRevision` includes the EA suffix to match the target version, while `#!productVersion` always uses `x.y.0`.

## Testing

Run the test suite:
```bash
python test_config_productversion.py
```

All tests verify:
1. Y-stream to Y-stream transitions
2. Y-stream to EA transitions
3. EA to Y-stream transitions
4. EA to EA transitions
5. Content preservation (only `#!productVersion` line is modified)

## Related Changes

This change works in conjunction with:
- **Charts tags handling** (`file_ops.py`): Adds version tags to charts files for Y-stream releases
- **Standard version replacements** (`rbc_release.py`): Updates version strings throughout Tekton/bundle/CSV files

### Charts RPA Files Tags

For **Y-stream releases only** (not EA), the charts RPA files (`*-charts-prod.yaml`, `*-charts-stage.yaml`) in konflux-release-data get THREE tags added to `spec.data.mapping.defaults.tags`:

1. `vX.Y` (e.g., `v3.5`)
2. `vX.Y.0-{{ release_timestamp }}` (e.g., `v3.5.0-{{ release_timestamp }}`)
3. `vX.Y.0` (e.g., `v3.5.0`)

**Example** (3.5-ea.2 → 3.5):
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

EA releases do **not** get these tags added.

## Why This Matters

### #!productVersion and #!scmRevision

The `#!productVersion` directive is used by the build system to:
- Set the product version in build artifacts
- Generate release metadata
- Track the release lineage

Using the `x.y.0` format (without EA suffixes) ensures consistency across EA and Y-stream builds for the same minor version.

The `#!scmRevision` directive points to the correct Git branch/tag for the source code, which must include EA suffixes when applicable.

### Charts Tags

The charts tags define which container image versions are promoted and made available:
- `vX.Y` — Latest stable version tag
- `vX.Y.0-{{ release_timestamp }}` — Timestamped release for tracking
- `vX.Y.0` — Specific patch version

These tags are only added for Y-stream releases to indicate GA availability.
