# dotnet-lateapexearlyspeed-jsonschema - Bowtie harness

A standalone [Bowtie](https://github.com/bowtie-json-schema/bowtie) test harness for
[`LateApexEarlySpeed.Json.Schema`](https://www.nuget.org/packages/LateApexEarlySpeed.Json.Schema).
It implements Bowtie's input/harness/output protocol (IHOP), accepting JSON-Lines commands on stdin
and writing responses to stdout.

Supported dialects:

- Draft 7
- Draft 2019-09
- Draft 2020-12

The library is consumed through NuGet (currently 5.0.0). The version under test is specified in the
[project file](dotnet-lateapexearlyspeed-jsonschema.csproj).

## Build and test

Prerequisites:

- .NET 8 SDK for native builds
- PowerShell for the fixture checks
- Docker with a running Linux container engine, or Podman
- Python 3.13+ and [Bowtie CLI](https://docs.bowtie.report/en/stable/)

The following examples use PowerShell on Windows and run from the repository root. Neither the library's source
repository nor an existing JSON Schema Test Suite checkout is required: .NET restores the library
from NuGet, and Bowtie downloads the official suite from GitHub.

### Python and Bowtie setup

Create a virtual environment once, then install Bowtie:

```powershell
python --version
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install bowtie-json-schema
```

In each new terminal, set the executable paths without activating the environment or modifying
global `PATH`:

```powershell
$python = (Resolve-Path .\.venv\Scripts\python.exe).Path
$bowtie = (Resolve-Path .\.venv\Scripts\bowtie.exe).Path
& $bowtie --version
dotnet --version
docker info --format '{{.OSType}}'
```

The container engine should report `linux`. Install the prerequisites and start Docker Desktop if
these checks fail. Network access to NuGet, PyPI, Microsoft's container registry and GitHub is needed.
A GitHub token is not normally required; GitHub API rate limits can require waiting or authenticating.
If Python or Bowtie has been upgraded, check its current Python requirements before recreating the
environment.

### Test scope

| Tests | Native .NET | Linux container |
|---|---|---|
| Bundled fixture transcripts | `test-fixtures.ps1` | Replay and compare with the checked native results below |
| Bowtie smoke | Not run natively in this guide | `bowtie smoke` |
| All supported dialects' official compliance cases | Not run natively in this guide | `bowtie site collect -V` |

This guide excludes all `optional/` tests and does not use a pre-existing offline suite. Native fixture
checks require only .NET and PowerShell, not Docker, Python or Bowtie. Full official compliance tests
run through Bowtie against the container.

### Native fixture checks

```powershell
dotnet build dotnet-lateapexearlyspeed-jsonschema.csproj --configuration Release
if ($LASTEXITCODE -ne 0) { throw "Native build failed." }
.\test-fixtures.ps1
```

[test-fixtures.ps1](test-fixtures.ps1) checks the bundled transcripts for meta-schema resolution,
custom vocabularies, format behavior and registry retrieval URIs across all supported dialects.
Registry checks cover absent, relative and differing absolute `$id` values.
It validates response counts, sequence IDs, result shapes
and expected validity values. Errors, skips and mismatches fail the check. These fixtures are
regression tests, not the complete official JSON Schema Test Suite.

### Container smoke test

```powershell
$image = "local/lateapexearlyspeed-jsonschema"
docker build -t $image .
if ($LASTEXITCODE -ne 0) { throw "Container build failed." }
& $bowtie info -i "image:$image" --format json
if ($LASTEXITCODE -ne 0) { throw "Container startup failed." }
& $bowtie smoke -i "image:$image" -f json
if ($LASTEXITCODE -ne 0) { throw "Container smoke test failed." }
```

The smoke test should succeed for all three supported dialects. Check that `bowtie info` reports the
library version in the project file. Rebuild after changes; do not assume an existing `latest` image
matches the current source.

### Container fixture checks

After the native fixture checker succeeds, replay the same transcripts through Docker and compare
every run response. Startup metadata is excluded from comparison because OS information differs.

```powershell
$ErrorActionPreference = "Stop"
foreach ($fixture in @("metaschema-test-cases.jsonl", "vocabulary-test-cases.jsonl", "format-test-cases.jsonl", "registry-test-cases.jsonl")) {
    $native = @(dotnet run --project dotnet-lateapexearlyspeed-jsonschema.csproj --no-build -c Release -- $fixture)
    if ($LASTEXITCODE -ne 0) { throw "Native replay failed: $fixture" }
    $container = @(Get-Content -LiteralPath $fixture | docker run --rm -i $image)
    if ($LASTEXITCODE -ne 0) { throw "Container replay failed: $fixture" }
    if ($native.Count -ne $container.Count) { throw "Response count mismatch: $fixture" }
    $nativeRuns = @($native | ForEach-Object { $_ | ConvertFrom-Json } |
        Where-Object { $_.PSObject.Properties.Name -contains "seq" } |
        ForEach-Object { $_ | ConvertTo-Json -Depth 100 -Compress })
    $containerRuns = @($container | ForEach-Object { $_ | ConvertFrom-Json } |
        Where-Object { $_.PSObject.Properties.Name -contains "seq" } |
        ForEach-Object { $_ | ConvertTo-Json -Depth 100 -Compress })
    if (($nativeRuns -join "`n") -cne ($containerRuns -join "`n")) {
        throw "Fixture results differ: $fixture`nNative: $nativeRuns`nContainer: $containerRuns"
    }
    Write-Output "${fixture}: native and container results match."
}
```

### Official compliance suite

The collection and completeness-check examples below expect the legacy suite layout
(`tests/<dialect>/` and `remotes/`). Upstream has migrated to compatibility-tagged cases in
`validation/tests/`, with per-case `externalSchemas`. Use a Bowtie release that supports that
layout when testing current upstream. An older collector can fail or test only remaining legacy
cases; pinning an older revision does not establish completeness for the current suite.

```powershell
$reports = Join-Path $PWD ("suite-reports-" + (Get-Date -Format "yyyyMMdd-HHmmss"))
& $bowtie site collect -i "image:$image" -V --output $reports
if ($LASTEXITCODE -ne 0) { throw "Official suite collection failed." }
```

This writes per-dialect reports for the supported dialects without publishing them. Use a new output
directory for each run. By default, Bowtie selects the official suite's latest `main` commit at or
before the start of the current UTC hour. Pass `--suite <commit>` to reproduce a particular revision.
Do not pass `--suite` for the normal online run. All three dialects use the same downloaded revision.

### Report summaries and completeness

Summarize failures, errors and skips separately from validity results:

```powershell
foreach ($name in @("draft7", "draft2019-09", "draft2020-12")) {
    & $bowtie summary --show failures --format json (Join-Path $reports "$name.json")
    if ($LASTEXITCODE -ne 0) { throw "Report summary failed: $name" }
}
```

A successful collection exit code means reports were generated, not that every test passed. Do not
compare current totals with a historical baseline: the suite can change.

The following Python standard-library check:

- Requires reports for all three dialects and the same suite commit.
- Downloads that exact official revision into memory, independently of the local filesystem.
- Compares every top-level source case's description, schema, test descriptions, instances and
  expected validity with the report, retaining duplicate-case counts. The official suite's `data`
  field is compared with Bowtie's corresponding `instance` field.
- Checks completion footers, unique/consecutive sequence IDs, one response per case, exact result
  lengths, Boolean validity values and the absence of errors, skips and mismatches.
It checks the saved Bowtie container reports without running the harness again. No local suite checkout
or native build is needed for this report check.

```powershell
@'
from collections import Counter
from io import BytesIO
import json
from pathlib import Path
import re
import sys
from urllib.request import urlopen
from zipfile import ZipFile

def require(condition, message):
    if not condition:
        raise RuntimeError(message)

directory = Path(sys.argv[1])
names = ("draft7", "draft2019-09", "draft2020-12")
reports = {}
for name in names:
    path = directory / (name + ".json")
    records = [json.loads(line) for line in path.read_text(encoding="utf-8-sig").splitlines() if line.strip()]
    require(len(records) >= 2, f"{name}: empty or incomplete report")
    require(records[-1] == {"did_fail_fast": False}, f"{name}: missing footer or early termination")
    reports[name] = records

commits = set()
for records in reports.values():
    href = records[0]["metadata"]["Commit"]["href"]
    match = re.fullmatch(r"https://github.com/json-schema-org/JSON-Schema-Test-Suite/commit/([0-9a-f]{40})", href)
    require(match is not None, f"Unexpected suite commit metadata: {href}")
    commits.add(match.group(1))
require(len(commits) == 1, "Dialect reports use different suite revisions")
commit = commits.pop()
print(f"Suite revision: {commit}")
url = f"https://codeload.github.com/json-schema-org/JSON-Schema-Test-Suite/zip/{commit}"
with urlopen(url, timeout=60) as response:
    archive = ZipFile(BytesIO(response.read()))

def signature(case):
    tests = [
        {
            "description": test["description"],
            "instance": test["instance"] if "instance" in test else test["data"],
            "valid": test["valid"],
        }
        for test in case["tests"]
    ]
    return json.dumps(
        {"description": case["description"], "schema": case["schema"], "tests": tests},
        sort_keys=True, separators=(",", ":"),
    )

def check_results(label, case, outcome):
    require("results" in outcome, f"{label}: error, skip or missing results: {outcome}")
    expected = [test["valid"] for test in case["tests"]]
    require(all(type(value) is bool for value in expected), f"{label}: non-Boolean expectations")
    actual = outcome["results"]
    require(type(actual) is list and len(actual) == len(expected), f"{label}: incorrect result count")
    for index, (result, wanted) in enumerate(zip(actual, expected)):
        require(type(result) is dict and set(result) == {"valid"}, f"{label}, test {index}: invalid/error/skipped result: {result}")
        require(type(result["valid"]) is bool, f"{label}, test {index}: non-Boolean validity")
        require(result["valid"] == wanted, f"{label}, test {index}: expected {wanted}, got {result['valid']}")
    return len(expected)

total = 0
for name, records in reports.items():
    header = records[0]
    require(len(header["implementations"]) == 1, f"{name}: expected one implementation")
    implementation = next(iter(header["implementations"]))
    cases, outcomes = {}, {}
    for record in records[1:-1]:
        seq = record["seq"]
        require(type(seq) is int, f"{name}: non-integer sequence")
        target = cases if "case" in record else outcomes
        require(seq not in target, f"{name}: duplicate sequence {seq}")
        target[seq] = record["case"] if "case" in record else record
    require(set(cases) == set(outcomes) == set(range(1, len(cases) + 1)), f"{name}: missing cases or responses")
    require(len(records) == 2 * len(cases) + 2, f"{name}: unexpected report records")
    source = []
    for member in archive.namelist():
        parts = member.split("/")
        if len(parts) == 4 and parts[1:3] == ["tests", name] and parts[3].endswith(".json"):
            source.extend(json.loads(archive.read(member)))
    require(source, f"{name}: no official source cases found")
    require(Counter(map(signature, source)) == Counter(map(signature, cases.values())), f"{name}: source/report cases differ")
    count = 0
    for seq, case in sorted(cases.items()):
        outcome = outcomes[seq]
        require(outcome["implementation"] == implementation, f"{name}: unexpected implementation")
        require(outcome["expected"] == [test["valid"] for test in case["tests"]], f"{name}: incorrect recorded expectations")
        count += check_results(f"{name} Docker seq {seq}", case, outcome)
    total += count
    print(f"{name}: {len(cases)} cases, {count} container tests passed; complete.")
archive.close()
print(f"All {total} official-scope container tests passed; no errors, skips, mismatches or missing responses.")
'@ | & $python - $reports
if ($LASTEXITCODE -ne 0) { throw "Report verification failed." }
```

Keep the reports and printed suite revision when sharing results. They are generated test output,
not source files to commit. None of these local commands publishes an image, release or website.

## Validation behavior

The bundled [meta-schemas](meta-schemas/) support resolution of references to canonical meta-schema
documents. Other external references resolve through the test case's registry.

Registry documents are passed unchanged to the library with their retrieval URI using
`ExternalSchemaRegistry.Register(schema, retrievalUri: baseUri)`. The harness does not inject or
overwrite `$id`: the library resolves relative references and `$id` values against the retrieval
URI while preserving a document's own canonical identifier.

Custom registry documents containing both `$vocabulary` and `$id` are registered as meta-schemas
before constructing the validator. Documents without `$id` remain available for ordinary reference
resolution but are not pre-registered as meta-schemas.

For standard dialects, `format` is an assertion in Draft 7 and annotation-only in Draft 2019-09 and
Draft 2020-12. Custom meta-schemas can select format-assertion behavior. A `false` vocabulary value
controls whether support is required; it does not disable assertions.

## CI and reports

- [build.yml](.github/workflows/build.yml) runs native fixture checks, then calls Bowtie's shared
  Harness CI to build `amd64` and `arm64` images and run smoke tests. Image publication is restricted
  to successful pushes to `main` in `bowtie-json-schema/dotnet-lateapexearlyspeed-jsonschema`.
  Published tags include the commit SHA, library version and `latest`. PRs and manual build runs do
  not publish.
- [report.yml](.github/workflows/report.yml) runs manually or every six hours, at minute 20 UTC, in
  the same official repository. It calls Bowtie's shared Harness Report to test the published image
  and update the rolling `report` release. Bowtie's central workflow collects reports separately.
  Report generation is independent of the image build workflow.

## Maintenance

[dependabot.yml](.github/dependabot.yml) checks Docker images daily and GitHub Actions weekly, with
a three-day cooldown and at most one open version-update PR per ecosystem. Major .NET SDK/runtime
image upgrades are excluded. Upgrade the target framework, container SDK/runtime images and CI SDK
version together in a reviewed PR. NuGet updates are manual.

After fixture and image checks succeed, same-repository Dependabot PRs targeting official `main` are
eligible for auto-merge, except PRs labelled `github_actions`. This requires repository auto-merge,
merge commits and workflow write permissions to be enabled; required reviews and branch rules still
apply.

A merge made using `GITHUB_TOKEN` does not trigger the normal push build. There is no separate
Dependabot publishing workflow, so a subsequent human-authenticated push to `main` is needed to
publish the updated image. Re-running an older workflow builds its original commit.

## License

[BSD 3-Clause](LICENSE).
