GitHub Action for Continuous Benchmarking of Pull Requests
=========================================

This repository provides a GitHub Action for continuous benchmarking of pull requests: it allows comparing the performance of the PR code against the base branch, aiming to detect possible performance regression. 

Concretely, this action collects data from the benchmark outputs generated by some popular tools, and when benchmark results get worse than the baseline exceeding the specified threshold, it can raise an alert via commit comment and/or workflow failure.

The following benchmarking tools are supported:

- [`cargo bench`][cargo-bench] for Rust projects
- `go test -bench` for Go projects
- [benchmark.js][benchmarkjs] for JavaScript/TypeScript projects
- [pytest-benchmark][] for Python projects with [pytest][]
- [Google Benchmark Framework][google-benchmark] for C++ projects
- [Catch2][catch2] for C++ projects

User-defined benchmark tools are also supported: this action accepts JSON files consisting of an array of benchmark results of specific format. See [Tool-specific setup](#tool-specific-setup) for more details.

## How to use: workflow setup

This action takes as input the files that contain benchmark outputs from the PR and base branch.
To ensure the measurements are carried out with similar conditions, the results should be collected on the same workflow and machine.

This is an example workflow that is triggered on pull requests targeting the master branch, and processes the benchmark results from JavaScript code. 

```yaml
name: Performance Regression Test

on:
  # This action only works for pull requests
  pull_request:
    branches: [master]

jobs:
  benchmark:
    name: Time benchmark
    runs-on: ubuntu-latest

    steps:
      # Check out pull request branch
      - uses: actions/checkout@v2
        with:
          path: pr
      # Check out base branch (to compare performance)
      - uses: actions/checkout@v2
        with:
          ref: master
          path: master
      - uses: actions/setup-node@v1
        with:
          node-version: '15'

      # Run benchmark on master and stores the output to a file
      - name: Run benchmark on master (baseline)
        run: cd master/examples/benchmarkjs && npm install && node bench.js | tee benchmarks.txt

      # Run benchmark on the PR branch and stores the output to a separate file (must use the same tool as above)
      - name: Run pull request benchmark
        run: cd pr/examples/benchmarkjs && npm install && node bench.js | tee benchmarks.txt

      - name: Compare benchmark result
        uses: openpgpjs/github-action-pull-request-benchmark@v1
        with:
          name: 'Time benchmark'
          # What benchmark tool the benchmarks.txt files came from
          tool: 'benchmarkjs'
          # Where the two output files from the benchmark tool are stored
          pr-benchmark-file-path: pr/examples/benchmarks.txt
          base-benchmark-file-path: master/examples/benchmarks.txt
          # A comment will be left on the latest PR commit if `alert-threshold` is exceeded 
          comment-on-alert: true
          alert-threshold: '130%'
          # Workflow will fail if `fail-threshold` is exceeded
          fail-on-alert: true
          fail-threshold: '150%'
          # A token is needed to leave commit comments
          github-token: ${{ secrets.GITHUB_TOKEN }}
```

By default, this action marks the result as performance regression when it is worse than the previous
exceeding a 50% threshold. For example, if the previous benchmark result was 100 iter/ns and this time
it is 150 iter/ns, it means 50% worse than the previous and an alert will happen. The threshold can
be changed by `alert-threshold` input.

For documentation about each action input, see the definitions in [action.yml](./action.yml).

### Tool-specific setup

Output files generated by the supported tools can be used directly with this action. See the `README.md` files under the `examples/` folder for details. Usually, it sufficies to take the stdout from a benchmark tool
and store it to file. Then, specify the corresponding file path in `pr-benchmark-file-path` or `base-benchmark-file-path`.

For any other benchmark tool, you will have to take care of formatting the benchmark results in a JSON file with appropriate structure:

**User-defined benchmark tools: compare pre-formatted JSON data**

The action supports comparing any properly formatted benchmark results by setting `tool: 'raw'`.

The benchmark files given to `pr-benchmark-file-path` and `base-benchmark-file-path` must be in JSON format and consist of an array of entries of the form:
```javascript
{
  name: string, // test identifier (must be unique)
  value: number, // numeric value to use for benchmark comparison
  unit: string, // unit of the `value`, for display only
  range: string, // complementary info about the value, for display only (e.g. stdev, confidence interval, etc)
  biggerIsBetter: boolean, // whether a bigger value is considered a better benchmark result
}
```

Alternatively, you can add a custom benchmark result parser to the action directly:

**How to add new benchmark tool support**

1. Add your tool name in `src/config.ts`
2. Implement the logic to extract benchmark results from the output in `src/extract.ts`


### Stability of Virtual Environment

Based on the benchmark results of the examples in this repository, the amplitude of the benchmarks
is about +- 10~20%. If your benchmarks use some resources such as networks or file I/O, the amplitude
might be bigger.

If the amplitude is not acceptable, please prepare a stable environment to run benchmarks.
GitHub action supports [self-hosted runners](https://help.github.com/en/actions/automating-your-workflow-with-github-actions/about-self-hosted-runners).


### Related actions

This is a hard fork of the [Continuous Benchmarking Github Action](https://github.com/rhysd/github-action-benchmark): it reuses some core components, but provides different features. In particular, there is no support for pushing benchmark results to Github Pages, and this action is not meant to carry out benchmark comparisons outside of pull requests.


[cargo-bench]: https://doc.rust-lang.org/cargo/commands/cargo-bench.html
[benchmarkjs]: https://benchmarkjs.com/
[pytest-benchmark]: https://pypi.org/project/pytest-benchmark/
[google-benchmark]: https://github.com/google/benchmark
[catch2]: https://github.com/catchorg/Catch2
