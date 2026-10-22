# model-validator v2 kit

Everything `itu-sdse-project-model-validator` needs to run and self-test
the new v2 model format (MLflow pyfunc export of the PyTorch CNN), as a
drop-in update. The validation *method* deliberately stays in the project
repository (`itu-sdse-project@v2`, `scripts/validate_model.py`); this action
only provides the plumbing.

## Contents

```
repo-files/
  action.yml                        drop-in replacement for the repo's action.yml
  .github/workflows/test_action.yml drop-in replacement for the self-test workflow
  kit/                              commit this directory as kit/ in the repo root
    sample-model/                  a v2-format model (MLflow pyfunc export, UNTRAINED)
    fixtures/expected.csv           fixture ground truth (see "Fixtures" below)
    fixtures/sample-model-expected.csv  the sample model's actual outputs on the fixtures
    fixtures/*.jpg                  the two known-fixture images
    validate_model.py               reference copy of the validation method
README.md                           this file
```

## How to apply

1. Replace `action.yml` with `repo-files/action.yml`. Legacy (v1 sklearn)
   behavior is fully preserved: every original step now runs only when
   `validate_command` is empty, so existing callers are unaffected.
2. Replace `.github/workflows/test_action.yml` with
   `repo-files/.github/workflows/test_action.yml`. It keeps the two legacy
   jobs and adds two: `v2-command-mode` (plumbing) and `v2-integration`
   (real pyfunc model + fixtures + the project's validation method).
3. Commit `repo-files/kit/` as `kit/` in the repo root.
4. Merge to `main` before the project repository's new validation workflow
   is pushed: itu-sdse-project pins `model-validator@main`, and until the
   update lands there, every push to it would run the legacy action and
   fail (the legacy checkout clobbers the caller's workspace and it looks
   for an in-run `model` artifact that does not exist).

## The v2 contract

- The calling workflow checks out its own repo, installs its own
  dependencies (torch, its package), and stages the model at `model_path`.
  In v2 the model is not pushed with the code - it is the pyfunc export a
  training run uploaded as a `model` artifact, so the caller fetches it
  with `gh run download` from the previous run (`download-artifact@v4`
  only sees the current run's artifacts).
- The action checks the model directory exists and runs
  `validate_command` in the caller's workspace. It does not check out
  anything (the v1 checkout replaced the caller's workspace) and installs
  nothing.
- The validation method is the caller's: for itu-sdse-project it is
  `python scripts/validate_model.py --model-dir models/model
  --fixtures-dir validation/fixtures`, which loads the pyfunc export,
  base64-encodes the fixture images (same contract as POST
  /invocations), and fails on any mismatch with `expected.csv`.

## Fixtures

- `kit/fixtures/expected.csv` is the ground truth (0 = clean,
  1 = defective) from the public test-set metadata; both labels were
  confirmed against the deployed Production model on 2026-10-18.
  Heads-up: its confidences on these fixtures are ~0.499 / ~0.506, so a
  retrained model may legitimately flip a fixture and fail validation.
- The sample model is UNTRAINED (random weights). It predicts
  "defective" for both fixtures, which is what
  `sample-model-expected.csv` records. It exists so the integration
  self-test can exercise the full path without a trained model; it is a
  plumbing test, not a model-quality test.
- Loading the sample model requires the `src` package from
  itu-sdse-project (the pyfunc wrapper imports `src.deployment.
  logging_wrapper`); the integration job installs it with
  `pip install -e itu-sdse-project`.

## Verified locally

- `validate_model.py` against the sample model: correct labels exit 0,
  a misclassification exits 1, missing model/fixtures fail with precise
  messages.
- The exact integration-job flow (staged `models/model`, fixtures with
  `sample-model-expected.csv`, the validate command above) passes with
  exit 0.
