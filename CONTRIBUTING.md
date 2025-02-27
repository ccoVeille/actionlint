# Reporting an issue

To report a bug, please submit a new ticket on GitHub. It's helpful to search similar tickets before making it.

https://github.com/rhysd/actionlint/issues/new

Providing a reproducible workflow content is much appreciated. If only a small snippet of workflow is provided or no
input is provided at all, such issue tickets may get lower priority since they are occasionally time consuming to
investigate.

# Sending a patch

Thank you for taking your time to improve this project. To send a patch, please submit a new pull request on GitHub.
It's helpful to check if a similar patch was rejected in the past before making it. actionlint focuses on detecting
mistakes so adding rules related to styles or conventions is basically not accepted.

https://github.com/rhysd/actionlint/pulls

Before submitting your PR, please ensure the following points:

- Confirm build/tests/lints passed. How to run them is described in the following sections.
- If you added a new feature, consider to add tests and explain it in [the usage document](docs/usage.md).
- If you added a new public API, consider to add tests and a doc comment for the API.

# Development

`make` (3.81 or later) is useful to run each tasks and reduce redundant builds/tests.

## How to build

```sh
go build ./cmd/actionlint
./actionlint -h
```

or

```sh
make build
```

`make build` generates some sources with `go generate`. When you want to avoid it, add `SKIP_GO_GENERATE=1` to `make` arguments.

Since actionlint doesn't use any cgo features, setting `CGO_ENABLED=0` environment variable is recommended to avoid troubles
around linking libc. `make build` does this by default.

## How to run tests

```sh
go test
```

or

```sh
make test
```

## How to run lint

[staticcheck](https://staticcheck.io/) is used to lint Go sources.

```sh
staticcheck ./...
```

or

```sh
make lint
```

## How to run fuzzer

Fuzz tests use [go-fuzz](https://github.com/dvyukov/go-fuzz). Install `go-fuzz` and `go-fuzz-build` in your system.

Since there are multiple fuzzing targets, `-func` argument is necessary. Specify a target which you want to run.

```sh
# Create first corpus
go-fuzz-build ./fuzz

# Run fuzzer
go-fuzz -bin ./actionlint_fuzz-fuzz.zip -func FuzzParse
```

or

```sh
make fuzz FUZZ_FUNC=FuzzParse
```

## How to release

When releasing v1.2.3 as example:

1. Ensure all changes were already pushed to remote by checking `git push origin master` outputs `Everything up-to-date`
2. Run `bash ./scripts/bump-version.bash 1.2.3`
3. Wait until [the CI release job](.github/workflows/release.yaml) completes successfully:
   - GoReleaser builds release binaries and make pre-release at GitHub and updates [Homebrew formula](./HomebrewFormula/actionlint.rb)
   - The CI job also updates version string in `./scripts/download-actionlint.bash`
4. Open the pre-release at [release page](https://github.com/rhysd/actionlint/releases) with browser
5. Write up release notes, uncheck pre-release checkbox and publish the new release
6. Run `changelog-from-release > CHANGELOG.md` locally to update [CHANGELOG.md](./CHANGELOG.md) and make a commit for the change
7. Run `git pull` to merge upstream changes to local `main` branch and run `git push origin main`
8. Update the playground by `./playground/deploy.bash` if it is not updated yet for the release

## How to generate the manual

`actionlint.1` manual is generated from [`actionlint.1.ronn`](./man/actionlint.1.ronn) by [ronn](https://github.com/rtomayko/ronn).

```sh
ronn ./man/actionlint.1.ronn
```

or

```sh
make ./man/actionlint.1
```

## How to develop playground

Visit [`playground/README.md`](./playground/README.md).

## How to deploy playground

Run [`deploy.bash`](./playground/deploy.bash) at root of repository. It does:

1. Ensure to install dependencies and to build `main.wasm`
2. Copy all assets to `./playground-dist` directory
3. Optimize `main.wasm` with `wasm-opt` which is a part of [Binaryen](https://github.com/WebAssembly/binaryen) toolchain
3. Switch branch to `gh-pages`
4. Move all files in `./playground-dist` to root of repository and add to repository
5. Make commit for deployment

```sh
# Prepare deployment
bash ./playground/deploy.bash
# Check it works fine by visiting localhost:1234
npm run serve
# If it looks good, deploy it
git push
```

## Maintain auto-generated sources

Some files are generated by scripts in [`scripts/`](./scripts) directory. These files are kept up-to-date by CI workflows.

### Maintain `popular_actions.go`

[`popular_actions.go`](./popular_actions.go) is a data set of metadata of popular actions hosted on GitHub. It is generated
automatically with `go generate`. The command runs [`generate-popular-actions`](./scripts/generate-popular-actions) script.

The script also can detect new major releases of popular actions on GitHub by giving `-d` flag.

The [`generate`](.github/workflows/generate.yaml) CI workflow weekly runs to detect new major releases and update
`popular_actions.go`. Runs can be found [here](https://github.com/rhysd/actionlint/actions/workflows/generate.yaml).

### Maintain `all_webhooks.go`

[`all_webhooks.go`](./all_webhooks.go) is a table all webhooks supported by GitHub Actions to trigger workflows. Note that
not all webhooks are supported by GitHub Actions.

It is generated automatically with `go generate` running [`generate-webhook-events`](./scripts/generate-webhook-events) script.

It fetches [`events-that-trigger-workflows.md`](https://github.com/github/docs/blob/main/content/actions/learn-github-actions/events-that-trigger-workflows.md),
parses the markdown document, and extracts webhook names and their types. For more details, see
[README.md at the script directory](./scripts/generate-webhook-events/README.md).

Updating `all_webhooks.go` is run weekly on CI by [`generate`](.github/workflows/generate.yaml) workflow.

### Maintain `actionlint-matcher.json`

[`actionlint-matcher.json`](.github/actionlint-matcher.json) is a matcher configuration to extract error annotations from outputs
of `actionlint` command. See [the document](docs/usage.md#problem-matchers) for its usage.

The regular expression is complicated because it can matches to outputs which contain ANSI color escape sequences. So the JSON
file is not modified manually.

It is generated by [`generate-actionlint-matcher`](./scripts/generate-actionlint-matcher) script. See the README.md file for the
usage of the script and how to run the tests for it.

### Maintain `availability.go`

[`availability.go`](./availability.go) is a table for conversion from workflow key (like `jobs.<job_id>.if`) to availability of
contexts and special functions. GitHub Actions limits contexts and functions in certain places. For example:

- limited workflow keys can access `secrets` context
- `jobs.<job_id>.if` and `jobs.<job_id>.steps.if` can use `always()` function.

`availability.go` is generated from [the contexts document](https://github.com/github/docs/blob/main/content/actions/learn-github-actions/contexts.md#context-availability)
using [generate-availability](./scripts/generate-availability) script. It is run through `go generate` in `rule_expression.go`.
See [the readme of the script](./scripts/generate-availability/README.md) for the usage of the script.

Update for `availability.go` is run weekly on CI by [`generate`](.github/workflows/generate.yaml) workflow.

## Testing

All tests are automated.

- Unit tests are implemented in `*_test.go` files for testing the corresponding APIs. Test data for unit tests are put in
  `testdata/` directory.
- UI tests based on matching to error messages are implemented in `linter_test.go` and all test data are stored in `testdata/`
  directory.
  - `testdata/examples/` contains tests for all examples in ['Checks' document](docs/checks.md). `*.yaml` files are an input
    workflow and `*.out` files are expected error messages.
  - `testdata/ok/` contains 'OK' tests. All workflow files in this directory should cause no errors.
  - `testdata/err/` contains 'Error' tests. Each `*.yaml` files are workflow inputs and corresponding `*.out` files are expected
    error messages (one error per line).
  - `testdata/projects/` contains 'Project' tests. Each directories represent a single project (meaning a repository on GitHub).
    Corresponding `*.out` files are expected error messages. Empty `*.out` file means the test case should cause no errors.
    'Project' test is used for use cases where multiple files are related (reusable workflows, local actions, config files, ...).
