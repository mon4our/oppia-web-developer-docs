## Introduction

Our CI process has a workflow, `check_ci_test_suites_to_run`, that determines which test suites to run for a given PR based on the files that the PR modified.
If your PR fails due to the `check_ci_test_suites_to_run` workflow, please look at the error log and depending on the error, do the following:

* If you see an error about the root files config, then see the [root files config section](#root-files-config) below.
* If you see an error about the test-to-modules mapping, then see the [test to modules mappings section](#test-to-modules-mappings) below.
* If your error doesn't match any of the above, please check the [overview section](#overview-of-partial-ci-tests-structure) below, which explains the structure of running partial CI tests.

## Overview of Partial CI Tests Structure

For non-python files, we run partial CI tests depending on the tests that a specific file affects. Running partial tests like this allows PRs to be checked faster using the tests that a PR actually modifies. This faster test speed allows developers to get feedback faster and frees up CI resources. 

The main structure of the partial CI tests code in our codebase is as follows:

* The script at `core/tests/test-dependencies/root-files-mapping-generator.ts` generates a root files mapping which maps the codebase's files with their root files. The generator does this by first generating a dependency graph which contains all files and their dependencies (through imports and Angular selectors). From there the root files mapping is generated by backtracking and finding which files depend on a certain file, until it reaches a root file (a file with no parent).
* The script at `core/tests/test-dependencies/test-to-modules-matcher.ts` is used to determine which Angular page modules a specific test depends on. This script exports a class called `TestToModulesMatcher` which is used by tests to register URLs that occur during that test through `TestToModulesMatcher.registerUrl`, this function matches the URL that is passed with an Angular page module and adds that to a list of collected Angular modules which represents the Angular page modules that have been collected during the test's runtime. From there the collected Angular modules are compared with its "golden file" counterpart at `core/tests/test-modules-mappings/{test-type}/{test-name}`, for example the acceptance test called `blog-admin/assign-roles-to-users-and-change-tag-properties` would have a golden file located at `core/tests/test-modules-mapping/acceptance/blog-admin/assign-roles-to-users-and-change-tag-properties.txt`. If any of the page modules that were collected are missing or are extra, an error will be outputted with the golden file that has the discrepancy and the page modules that are missing/extra.
* The script at `scripts/check_ci_test_suites_to_run.py` is used to check the CI test suites that need to run in a given PR. It does this by checking the `git diff` between the PR `HEAD` branch and the branch that is being merged into. It uses the above "root-files generator" and "test-modules mappings" to determine which tests to run and outputs it to the github workflow.

The process of determining which tests to run goes like this:

1. Generate a root files mapping which maps files to their root files. This file is generated at the start of every PR check run and is stored at `root-files-mapping.json`.
2. Run the script, `scripts/check_ci_test_suites_to_run.py` inside the Github workflow which will map changed files in a PR to their root files using the mapping above. Afterwards, map those root files to tests by using the golden files containing the page modules that a test uses.

# Test to Modules Mappings

The test-to-modules mappings at `core/tests/test-modules-mappings` contain the "golden" mappings of a test and the Angular page modules that they use. This golden file is validated everytime a test is run and if there is a discrepancy between the generated test-to-modules mapping and the golden file mapping, then it will output an error with the Angular page modules that are extra/missing. This ensures that the "golden" mappings are kept up to date whenever a test is created or modified.
If you get an error about the test-to-modules mapping, please do the following:

* If the error only happens on a few tests, please simply follow the error message and change the golden file with the Angular page modules that the error message shows.
* If the error happens on many tests, please consider adding it to the `COMMON_MODULES_TO_EXCLUDE` constant at `core/tests/test-dependencies/test-to-modules-matcher.ts`, with a corresponding glob of the CUJ that will cover that module, or a TODO if that CUJ doesn't exist yet. 

# Root Files Config

The root files config at `core/tests/root-files-config.json` contains all the valid root files that are generated in the root files mapping. A root file is a file in the dependency graph which has no parents, the most common root files in the codebase are page modules and other files like `README.md`. The "root files mapping generator" validates the `root-files-config.json` by comparing the PR's root files in the mapping with the root files found in the config. If there are any discrepancies, the generator will error; this ensures that the "root-files config" is always up to date with every PR.

If you get an error that there is a root file not contained in these lists, please add them to one of these lists depending on the context of the file:

* If the specific file has a chance to break some part of the Oppia website, like `src/index.html`, please add it to the `RUN_ALL_TESTS_ROOT_FILES` in the `root-files-config.json` file.
* If it is a file like `README.md` which doesn't affect any test, please add it to the `RUN_NO_TESTS_ROOT_FILES` in the `root-files-config.json` file.