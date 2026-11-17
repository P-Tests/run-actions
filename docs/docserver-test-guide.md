# Running the DocServer test workflow

Reference for the **Puppeteer DocServer test** workflow form (*Actions* → *Run workflow*).
Each field is described with its purpose and effect. Sections follow the order of the form.

The key idea: the **stand** selects the target server (the URL), and the build that server
runs selects the test code - the `Dep.Tests` tag of the same version. The **branch** field
overrides that choice, so any branch can be run against any stand without editing a config.

## 1. Use workflow from (`Branch:` selector)

A native GitHub control, not a form input. It selects the branch of the `run-actions`
repository used for the workflow definition and the composite actions (`.github/actions/...`).

This is the run-infrastructure branch, not the branch containing the tests. Leave it on
`master` unless you are testing changes to the workflows or actions themselves.

## 2. Stand to test (required)

Dropdown of stands: `kim`, `doc-linux` or `saul`. The stand selects the **target server**. Its URL
is taken from a repository secret:

| Stand       | Server     | Secret         |
| ----------- | ---------- | -------------- |
| `kim`       | release    | `URL_RELEASE`  |
| `doc-linux` | hotfix     | `URL_HOTFIX`   |
| `saul`      | teamlab    | `SAUL_TEAMLAB` |

The selected URL is written into the OS config (`testOptions.url` in `config_chrome_*.json`)
before the run, so both the tests and the cleanup target that server.

The stand also sets the **default** `Dep.Tests` ref (used only when the **branch** field is
empty). The *Resolve branch* job reads the build the stand serves - `VersionNum="10.0.0.169"`
in `web-apps/apps/documenteditor/main/app.js` - and checks out the `Dep.Tests` tag of the same
release (`10.0.0`) whose build is not lower than the server's: `v10.0.0.169` when it exists,
else the lowest later build of that release (`v10.0.0.172`, with a notice). A lower build or
another release is never taken. The chosen tag is shown in the run summary.

The job fails instead of guessing when the stand does not answer, when `app.js` has no
`VersionNum`, or when `Dep.Tests` has no such tag. In the last case tag the tests for the
build, or set the **branch** field.

To run a branch against a different server, leave the stand set to that server and fill in
the **branch** field (section 5).

The secrets `URL_RELEASE` and `URL_HOTFIX` must be defined in the `run-actions` repository
settings.

## 3. Operating system to run on (required)

`windows`, `linux` (the default), or `macos`. Determines which runners the tests execute on:
one run, one report, on that operating system only. There is no option for all three at once
- the full set on three systems is too heavy for one run; to cover several, start the workflow
once per system.

## 4. Tests to run (required)

`all` (the default), `desktop` or `mobile` - the UI type whose tests run:

| Value     | Runs                           | Launch name                   | `uiType`  |
| --------- | ------------------------------ | ----------------------------- | --------- |
| `all`     | every test                     | `Github Action All Tests`     | `all`     |
| `desktop` | the `tests/web/desktop` folder | `Github Action Desktop Tests` | `desktop` |
| `mobile`  | the `tests/web/mobile` folder  | `Github Action Mobile Tests`  | `mobile`  |

The choice is shown in three places, so the runs of one kind are told from the others:

- the name of the workflow run on the *Actions* tab
  (`Puppeteer DocServer test - mobile tests - kim - linux`) and the Telegram message;
- the name of the ReportPortal launch (unless section 10 sets another);
- the `uiType` attribute of the launch - filter the launches by `uiType:mobile`, or build a
  widget on it, whatever the launch is named.

With a test path (section 6) the path decides: `uiType` is the type of the folder it is in
(`mobile` for `tests/web/mobile/word`), and the run fails in *Prepare Test Matrix* when the
path belongs to another type than the one chosen here. Leave `all` when a path is set.

A type whose folder the `Dep.Tests` ref does not have fails in *Prepare Test Matrix* too.

## 5. Dep.Tests branch override (optional)

The name of a `Dep.Tests` branch or tag to use instead of the stand default.

- Empty: the tag of the build the stand runs is used (see section 2).
- Set (for example `feature/my-tests`): the workflow switches to that branch, ignoring the
  stand default.

The server is still determined by the stand, not by the branch. This is what allows the
same branch to be run against release and hotfix in turn.

## 6. Specific test file or directory (optional)

A path to a single test or folder inside `tests/`.

- Empty: the tests chosen in section 4 run.
- Set (for example `tests/web/desktop/word/hometab/font`): only that file or folder runs, and
  the launch is named `Github Action Target Tests`.

## 7. Tests per parallel chunk (default `100`)

The chunk size — the number of tests in one parallel batch. All discovered tests are split
into chunks of this size, and each chunk runs as a separate parallel job. A smaller value
produces more chunks, reducing wall-clock time but using more runners. If there are fewer
tests than the chunk size, a single chunk runs.

## 8. Clear test files from the remote server after each chunk (default on)

Controls cleanup of the files the tests create on the remote server.

- On: after each chunk, `clearRun` deletes the test files from the stand.
- Off: the files remain (useful for debugging, but accumulates data on the stand).

Cleanup targets the same server selected by the stand. Keep this enabled for regular runs.

## 9. ReportPortal project name (default `docsserver`)

The ReportPortal project that receives the results.

- The shared team project is `docsserver` (the default).
- A personal project uses the format `name_personal` (for example `imuminov_personal`).
  Use it to send results to your own project instead of the shared one.

## 10. ReportPortal launch name (optional)

The name of the launch in ReportPortal.

- Empty: the name follows the tests of the run - `Github Action All Tests`,
  `Github Action Desktop Tests` or `Github Action Mobile Tests` (section 4), and
  `Github Action Target Tests` with a test path (section 6).
- Set: the launch is named as entered. Its `uiType` attribute still tells the tests it holds.

The auto-analyzer (section 12) compares a launch with the earlier launches of the same name,
so each name keeps a history of its own.

## 11. ReportPortal API key (optional)

The ReportPortal access key.

- Empty: the repository secret `REPORT_KEY` is used (the normal mode).
- Set: overrides the secret for this run.

Leave empty unless a different key is required.

## 12. Run the ReportPortal auto-analyzer (default on)

Starts the ReportPortal auto-analyzer as soon as the launch is complete, so the failures do
not have to be analyzed by hand. The analyzer compares them with the history of launches
with the same name and assigns the known defect types automatically.

It runs inside the job that completes the launch: the *Merge ... Launches* job right after
the merge, or - when the whole run fits into a single chunk and there is nothing to merge -
the chunk job itself, after the tests. The job only starts the analysis; ReportPortal
performs it in the background, so nothing waits for the result.

The settings match the defaults of the *Analyze Launches* dialog in ReportPortal: the base is
*Current and all previous launches with the same name*, and all three kinds of items are
analyzed (to investigate, analyzed automatically, analyzed manually). The rest - similarity
thresholds, indexed log lines - comes from the project's *Settings - Auto-Analysis* page.

- On: every complete launch is sent to the analyzer.
- Off: launches are left unanalyzed; they can still be analyzed manually in ReportPortal.

A failure to start the analysis is reported as a warning and does not fail the run.

## Typical run

1. **Use workflow from**: `master`.
2. **Stand to test**: the target server (`kim` for release, `doc-linux` for hotfix).
3. **Operating system**: `linux`, or another single operating system.
4. **Tests to run**: `all`, or `desktop` / `mobile` for the tests of one UI type.
5. Leave the remaining fields at their defaults.
6. **Run workflow**.

To run a specific branch against both servers, set **branch** to your branch and run the
workflow twice — once with `kim`, once with `doc-linux`.

Per-chunk status appears on the *Actions* tab, results appear in ReportPortal, and the
overall outcome is sent to Telegram.

## Several runs at once

One run of this workflow executes at a time. A run started while another is in progress waits
for it (*Pending* on the *Actions* tab), and so does every run started after: they form a
queue of up to 100 runs and start one by one, in the order they began to wait. Nothing is
cancelled, so the desktop and the mobile tests, several stands or several branches can be
started in a row and left to run.

To drop a run from the queue, cancel it on the *Actions* tab.

## Required fields

Fields marked with `*` (**Stand to test**, **Operating system**, **Tests to run**) are required.
They have default values, so pressing *Run workflow* without changing them is sufficient.

## What each selector controls

| Selector                       | Selects               | Controls                                          |
| ------------------------------ | --------------------- | ------------------------------------------------- |
| **Use workflow from** (top)    | `run-actions` branch  | workflow code, actions                            |
| **Stand**                      | target server         | server URL (from secret) and, by its build, the default `Dep.Tests` tag |
| **branch** field               | `Dep.Tests` ref       | the test code (overrides the stand default)       |
| **Tests to run**               | UI type               | the folder of tests that runs, the launch name and its `uiType` attribute |

## Daily and Hourly workflows

The **Daily** and **SaaS Hourly** workflows are structured similarly, with two differences:

- There is no Stand field and no branch override. Several providers are tested at once
  (docspaceEU, docspaceUS, ncoc, ocis), each with the `Dep.Tests` tag chosen by the build its
  own document server runs - the same rule as a stand (section 2). The document server is
  `testOptions.apiUrl` of the provider's config in `Dep.Tests` (read from `master`);
  `.github/providers.json` names that config for each provider. A provider whose build has
  no tag fails in *Resolve branches*, and its runs do not start.
- They run on a schedule (cron) and can also be started manually via *Run workflow* with
  the same fields (operating system, test path, chunk size, clean, ReportPortal fields).
- The auto-analyzer runs there as well, always on, in the same job that completes the launch.
- There is no **Tests to run** field and no `uiType` attribute: they run the `daily` / `hourly`
  suite of each provider, a fixed list of tests, not a folder.
- They share one concurrency group without a queue: at most one run waits, and a newer one
  replaces it - a scheduled run that is late is dropped for the next one instead of piling up.
