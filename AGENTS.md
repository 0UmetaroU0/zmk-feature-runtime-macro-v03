This repository contains a ZMK module.

## Initialization (first time only)

This repo is created from template. Below template code should be changed when initially starting implementation from template.

- README.md
  - Remove Summary, "Usage" and "More Info" which is for template users. Replace to proper module description.
  - Update "Module User Guide" properly
  - Update CI status badge to link to your repo instead of template
- zephyr/module.yml: Change module name properly.
- rename artifact defined in tests/zmk-config/build.yaml properly
- remove this "Initialization" section from AGENT.md after confirming all items accomplished.

## Dev Rules

- Ensure pre-commit works and never bypass pre-commit check.
- Write simple and sufficient tests for new features.
  - Unit test: Test major functionalities as much as possible by adding case to `tests/<test case>`.
    You might have to add test only logic like executing logic at zephyr initialization to improve test coverage.
  - Build test: Enable feature in `tests/zmk-config/*`. It is to verify build works for real device and to easily test with real device.
    Ensure the feature and device is enabled as expected in the build by verifying output in `test.py`.
- Update README.md properly to guide how to use the module to unfamiliar ZMK keyboard users. Keep the guide simple but sufficient!

## Commands

Test command usually takes 1min.

```
# Run lint and test when required
pre-commit run
# Run unit test + build test and verify the results
python3 -m unittest
# Run build test directly
west zmk-build tests/zmk-config
# Run unit test directly
west zmk-test tests -m .
```
