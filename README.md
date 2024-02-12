# Yarn Workspaces Listing Action

[![Test Status](https://img.shields.io/github/actions/workflow/status/zetavg/yarn-workspaces-list-action/test-all.yml?style=flat-square&label=test&cacheSeconds=300&link=https%3A%2F%2Fgithub.com%2Fzetavg%2Fyarn-workspaces-list-action%2Factions%2Fworkflows%2Ftest-all.yml)](https://github.com/zetavg/yarn-workspaces-list-action/actions/workflows/test-all.yml)

This GitHub action lists all Yarn (v2+) Workspaces that match a specific condition.

## Use Cases

* Running tests in parallel across all packages with a "test" script defined in their package.json using dynamic matrixes. ([example](https://github.com/zetavg/yarn-workspaces-list-action/blob/main/.github/workflows/test-example-test-all-packages-in-parallel.yml#L56-L58))
* Run a particular command across all workspaces that match a specific condition with dynamic matrixes.

![](https://github.com/zetavg/yarn-workspaces-list-action/assets/3784687/5c812ff1-ce4e-4fc3-afe4-5d2c16b59c92)

## Inputs

- `condition`: A shell script condition to match against the workspaces. For example, to match all workspaces that have a "test" script defined in their package.json, you can use: `[ -f "$workspace_path/package.json" ] && jq -e ".scripts.test" "$workspace_path/package.json"`. Default is `true`, which will list all workspaces except the root workspace.
- `workspace-root`: The root directory of the Yarn workspace. Default is `.`, which will be the root of the repository.

## Outputs

- `workspace-paths`: A list of matched yarn workspace paths.

## Example Usage

In your workflow file, you can use this action as follows:

```yml
- name: List workspaces with tests
  uses: zetavg/yarn-workspaces-list-action@v2
  id: list-workspaces
  with:
    condition: '[ -f "$workspace_path/package.json" ] && jq -e ".scripts.test" "$workspace_path/package.json"'
```

This will list all workspaces that have a "test" script defined in their `package.json`, and set the paths to these workspaces as an output that can be used in subsequent steps or jobs.

Here's an example of how you can use the output in a subsequent job, to run tests in parallel across all workspaces with a "test" script defined in their `package.json`:

```yml
jobs:
  # Define a job that lists all workspaces with a test script.
  # This job will be needed by the test job to generate the matrix of workspaces to test dynamically.
  prepare-test:
    runs-on: ubuntu-latest
    outputs:
      # Expose the workspace path output from the list-workspaces step for other jobs to use it.
      workspace-paths: ${{ steps.list-workspaces.outputs.workspace-paths }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 1

      # Yarn is required for the yarn-workspaces-list-action to work.
      # To do this, we'll install Node.js and enable Corepack for picking up the correct yarn version here. This may vary depending on your project.
      # Note that running `yarn install` is not necessary.
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version-file: '.node-version'
      - name: Enable Corepack
        run: corepack enable

      # Use the yarn-workspaces-list-action to list all workspaces that have a test script.
      - name: List workspaces with tests
        uses: zetavg/yarn-workspaces-list-action@v2
        id: list-workspaces
        with:
          # A condition that checks if the package has a "test" script defined in its package.json.
          condition: '[ -f "$package_path/package.json" ] && jq -e ".scripts.test" "$package_path/package.json"'
          workspace-root: . # Optional, defaults to the root of the repository (".").

  # Define a job that uses the matrix strategy to test all workspaces in parallel.
  test:
    needs:
      - prepare-test # This job needs the prepare-test job to provide the package paths that need to be tested.
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false # Allow all jobs to complete even if one or more fails.
      matrix:
        # Use the package paths from the prepare-test job to create a matrix of workspaces to test.
        dir: ${{ fromJson(needs.prepare-test.outputs.workspace-paths) }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version-file: '.node-version'
      - name: Enable Corepack
        run: corepack enable
      - name: Install dependencies
        run: yarn install
      - name: Test
        run: |
          cd ${{ matrix.dir }}
          yarn test
```

For more details, see this [example workflow](https://github.com/zetavg/yarn-workspaces-list-action/blob/main/.github/workflows/test-example-test-all-packages-in-parallel.yml) ([workflow runs](https://github.com/zetavg/yarn-workspaces-list-action/actions/workflows/test-example-test-all-packages-in-parallel.yml)), which runs tests in parallel across all packages in [this example yarn workspace](https://github.com/zetavg/yarn-workspaces-list-action/tree/main/examples/test-all-packages-in-parallel).

## License

This project is licensed under the MIT License. See the LICENSE file for details.
