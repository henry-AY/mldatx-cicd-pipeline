# MATLAB Test Runner Guide

This document explains how the MATLAB portion of the CI/CD pipeline works. It covers:

* How MATLAB is launched from Python
* How test suites are located
* How `.mldatx` files are executed
* How results are exported
* How MATLAB returns status codes
* How failures, crashes, and exceptions are handled

This guide is for maintainers who may need to modify or debug MATLAB-side behavior.

---

## 1. Overview

The MATLAB Test Runner is the middle stage of the CI/CD pipeline.
It is invoked by Python using:

```
matlab -batch "run('<path>/helper.m')"
```

MATLAB is **headless** (no GUI, no popups), ensuring that CI/CD never gets blocked by dialogs or model windows.

MATLAB does the following:

1. Reads environment variables passed by Python
2. Locates the correct subsystem folder
3. Finds the newest `.mldatx` test suite
4. Runs all tests using `sltest.testmanager`
5. Exports results to a timestamped output file
6. Cleans up open models & Test Manager state
7. Exits MATLAB using numerical return codes

---

## 2. Files Involved

MATLAB uses two important files:

#### ✔ `helper.m`

* Entry point for all MATLAB execution
* Validates environment variables
* Calls `automatedTestingScript.m`
* Returns crash code `-99` if anything unexpected happens

#### ✔ `automatedTestingScript.m`

* Loads configuration JSON
* Locates subsystem folder
* Finds the newest `.mldatx` test file
* Runs tests via Simulink Test Manager
* Exports results to `<results_folder>/<timestamp>_results.mldatx`
* Determines whether tests passed or failed
* Returns:

  * `0` → Passed
  * `1` → Failed

Both rely on the **Simulink Test Manager API**:

```
sltest.testmanager
sltest.testmanager.run
sltest.testmanager.exportResults
sltest.testmanager.clear
sltest.testmanager.close
```

---

## 3. Environment Variables (Set by Python)

MATLAB does **not** get command-line arguments.
Instead, Python injects information using environment variables:

| Variable         | Purpose                                       |
| ---------------- | --------------------------------------------- |
| `TEST_SUBSYSTEM` | Which folder to test                          |
| `BASE_PATH`      | Root directory containing all project folders |
| `CONFIG_FILE`    | Path to `config.json`                         |

MATLAB reads these using:

```matlab
subsystem_to_test = getenv('TEST_SUBSYSTEM');
BASE_PATH = getenv('BASE_PATH');
CONFIG_FILE = getenv('CONFIG_FILE');
```

If any are missing, MATLAB throws a controlled error.

---

## 4. Folder Layout Expected

Every project folder must contain:

```
<BASE_PATH>/<subsystem>/  
    TestSuites/      → contains .mldatx test files  
    TestResults/     → output directory for exported results  
```

These names (**TestSuites**, **TestResults**) come from `config.json`.

---

## 5. Test Selection Logic

MATLAB chooses **the newest** `.mldatx` file in the subsystem’s TestSuites folder:

```matlab
allFiles = dir(fullfile(testSuitesDir, '*.mldatx'));
[~, idx] = sort([allFiles.datenum], 'descend');
latestFile = allFiles(idx(1));
```

This ensures teams don’t need to update filenames for every test revision.

---

## 6. Running Tests

Tests are executed using Simulink Test Manager:

```matlab
sltest.testmanager.TestFile(pathToTest);
results = sltest.testmanager.run;
```

The structure `results` has an `Outcome` property:

* `"Passed"`
* `"Failed"`

MATLAB logs the outcome and prints details to stdout (captured by Python).

---

## 7. Results Export

Results are exported to a timestamped snapshot:

```
<TestResults>/<testname>_YYYY_MM_DD_HHMMSS_results.mldatx
```

Code:

```matlab
timestamp = datestr(now, 'yyyy_mm_dd_HHMMSS');
resultFile = fullfile(resultsDir, sprintf('%s_%s_results.mldatx', baseName, timestamp));
sltest.testmanager.exportResults(results, resultFile);
```

Maintainers can open these snapshots in MATLAB’s Test Manager GUI.

---

## 8. Cleanup

MATLAB performs an aggressive cleanup to avoid memory leaks or lingering models that would crash future runs:

```matlab
sltest.testmanager.clear;
sltest.testmanager.close;

bdCloseAll = find_system('type', 'block_diagram');
for k = 1:length(bdCloseAll)
    close_system(bdCloseAll{k}, 0);
end
```

This ensures every CI run starts fresh.

---

## 9. MATLAB Return Codes

MATLAB returns integer codes that Python interprets:

| MATLAB Return Code | Meaning                  | Behavior                         |
| ------------------ | ------------------------ | -------------------------------- |
| `0`                | All tests passed         | Python marks subsystem as PASSED |
| `1`                | One or more tests failed | Python marks subsystem as FAILED |
| `-99`              | MATLAB crash/exception   | Python marks subsystem as ERROR  |

MATLAB produces `-99` only when:

* Unexpected exceptions occur
* Bad environment variables
* Something fails *outside* normal test execution

---

## 10. Error Handling

#### Handled test errors (normal test failures)

MATLAB still exports results.
Return code = `1`.

#### Unhandled MATLAB exceptions (crashes)

MATLAB prints the stack trace and exits with:

```matlab
exit(-99);
```

Python logs stderr and marks the subsystem as **ERROR**.

---

## 11. helper.m (Flow Summary)

1. Validate environment
2. Run automatedTestingScript
3. On success → return test result code
4. On crash → return `-99`

Flow:

```matlab
try
    status = automatedTestingScript(...);
    exit(status);
catch e
    fprintf('[ERROR] Exception occurred:\n%s\n', e.getReport());
    exit(-99);
end
```

This wrapper ensures Python is *never* left hanging.

---

## 12. automatedTestingScript.m (Flow Summary)

1. Load JSON config
2. Resolve subsystem paths
3. Locate the newest test suite
4. Run tests via Test Manager
5. Export results
6. Print logs
7. Cleanup Test Manager + models
8. Return `0` / `1`

---

## 13. Troubleshooting Guide

#### **MATLAB never finishes running**

* Usually Simulink build step is stuck
* Check if a pop-up is blocking execution
* Review full MATLAB stdout in the logs

#### **No `.mldatx` detected**

* Ensure the folder name matches `config.json`
* Ensure the test suites directory exists:
  `TestSuites/`

#### **Results not generated**

* Check write permissions for:
  `<subsystem>/TestResults`

#### **MATLAB returns -99 immediately**

* Typically caused by:

  * Missing environment variables
  * Invalid config.json path
  * Base path not found
