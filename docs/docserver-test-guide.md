# Running the DocServer test workflow

Reference for the **Puppeteer DocServer test - Desktop** workflow form (*Actions* → *Run workflow*).
Each field is described with its purpose and effect. Sections follow the order of the form.

This workflow runs the **desktop** tests. The mobile tests have a workflow of their own with
the same form less one field - see [Mobile tests](#mobile-tests).

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
the **branch** field (section 4).

The secrets `URL_RELEASE` and `URL_HOTFIX` must be defined in the `run-actions` repository
settings.

## 3. Operating system to run on (required)

`windows`, `linux` (the default), `macos`, or `all`. Determines which runners the tests execute
on.

- One system: one launch in ReportPortal, on that operating system only.
- `all`: windows and linux run **one after another** - windows first, then linux - never at
  the same time, so the stand is loaded by one system only. Each system reports into a launch
  of its own: same name, its own `platform` attribute, its own Telegram message. Linux starts
  when the windows chunks are over, whatever their result: failed tests on windows do not stop
  linux. The run takes about twice as long as one system.
- `macos` runs only when chosen on its own; it is not part of `all`, and the ReportPortal
  dashboards have no macOS columns - its launches are seen in the launch lists only.

Cancelling the run on the *Actions* tab stops the systems that have not started as well.

## 4. Dep.Tests branch override (optional)

The name of a `Dep.Tests` branch or tag to use instead of the stand default.

- Empty: the tag of the build the stand runs is used (see section 2).
- Set (for example `feature/my-tests`): the workflow switches to that branch, ignoring the
  stand default.

The server is still determined by the stand, not by the branch. This is what allows the
same branch to be run against release and hotfix in turn.

## 5. Specific test file or directory (optional)

A path to a single test or folder inside `tests/web/desktop`.

- Empty: every test of the `tests/web/desktop` folder runs.
- Set (for example `tests/web/desktop/word/hometab/font`): only that file or folder runs, and
  the launch is named `Github Action Target Tests`.

A path outside `tests/web/desktop` - a mobile test, or `tests/web` itself - fails in *Prepare
Test Matrix*: the mobile tests are run by their own workflow ([Mobile tests](#mobile-tests)).
So does a `Dep.Tests` ref that has no `tests/web/desktop` folder.

## 6. Tests per parallel chunk (default `100`)

The chunk size — the number of tests in one parallel batch. All discovered tests are split
into chunks of this size, and each chunk runs as a separate parallel job. A smaller value
produces more chunks, reducing wall-clock time but using more runners. If there are fewer
tests than the chunk size, a single chunk runs.

## 7. Clear test files from the remote server after each chunk (default on)

Controls cleanup of the files the tests create on the remote server.

- On: after each chunk, `clearRun` deletes the test files from the stand.
- Off: the files remain (useful for debugging, but accumulates data on the stand).

Cleanup targets the same server selected by the stand. Keep this enabled for regular runs.

## 8. ReportPortal project name (default `docsserver`)

The ReportPortal project that receives the results.

- The shared team project is `docsserver` (the default).
- A personal project uses the format `name_personal` (for example `imuminov_personal`).
  Use it to send results to your own project instead of the shared one.

## 9. ReportPortal launch name (optional)

The name of the launch in ReportPortal.

- Empty: `Github Action Desktop Tests`, and `Github Action Target Tests` with a test path
  (section 5).
- Set: the launch is named as entered.

Whatever its name, the launch has the attribute `uiType:desktop`: filter the launches by it, or
build a widget on it. The *DocServer - All tests* dashboard in ReportPortal puts the desktop and
the mobile launches together.

The auto-analyzer (section 11) compares a launch with the earlier launches of the same name,
so each name keeps a history of its own.

## 10. ReportPortal API key (optional)

The ReportPortal access key.

- Empty: the repository secret `REPORT_KEY` is used (the normal mode).
- Set: overrides the secret for this run.

Leave empty unless a different key is required.

## 11. Run the ReportPortal auto-analyzer (default on)

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
3. **Operating system**: `linux`, another single system, or `all` for windows and linux in a row.
4. Leave the remaining fields at their defaults.
5. **Run workflow**.

To run a specific branch against both servers, set **branch** to your branch and run the
workflow twice — once with `kim`, once with `doc-linux`.

Per-chunk status appears on the *Actions* tab, results appear in ReportPortal, and the
overall outcome is sent to Telegram.

## Mobile tests

The **Puppeteer DocServer test - Mobile Web** workflow runs the `tests/web/mobile` folder. Its form
is the one described above without the **Operating system** field: the mobile tests run in a
device Chrome emulates, the system of the runner changes nothing in them, so they always run
on linux - one launch per run.

What differs from the desktop workflow:

|                        | Desktop                              | Mobile Web                              |
| ---------------------- | ------------------------------------ | --------------------------------------- |
| Workflow               | *Puppeteer DocServer test - Desktop* | *Puppeteer DocServer test - Mobile Web* |
| Tests                  | `tests/web/desktop`                  | `tests/web/mobile`                      |
| Operating system       | the one chosen, or windows + linux   | linux                                   |
| Launch name            | `Github Action Desktop Tests`        | `Github Action Mobile Tests`            |
| `uiType` of the launch | `desktop`                            | `mobile`                                |

The name of a run on the *Actions* tab starts with the name of its workflow
(`Puppeteer DocServer test - Mobile Web - kim`), and the Telegram message has it in *Workflow*.

The other fields - stand, branch, chunk size, clean, the ReportPortal fields, the analyzer -
work as in sections 2 and 4-11. The test path has to be inside `tests/web/mobile`; with one the
launch is named `Github Action Target Tests`, as in the desktop workflow.

To run both kinds of tests, start both workflows: the second run waits for the first.

## Several runs at once

One run of the two workflows - desktop and mobile - executes at a time: they load the same
stands and share one queue. A run started while another is in progress waits for it (*Pending*
on the *Actions* tab), and so does every run started after: they form a queue of up to 100 runs
and start one by one, in the order they began to wait. Nothing is cancelled, so the desktop and
the mobile tests, several stands or several branches can be started in a row and left to run.
Several operating systems need one run only: `all` in section 3.

To drop a run from the queue, cancel it on the *Actions* tab.

## Required fields

Fields marked with `*` (**Stand to test**, **Operating system**) are required.
They have default values, so pressing *Run workflow* without changing them is sufficient.

## What each selector controls

| Selector                       | Selects               | Controls                                          |
| ------------------------------ | --------------------- | ------------------------------------------------- |
| **Use workflow from** (top)    | `run-actions` branch  | workflow code, actions                            |
| **Stand**                      | target server         | server URL (from secret) and, by its build, the default `Dep.Tests` tag |
| **branch** field               | `Dep.Tests` ref       | the test code (overrides the stand default)       |
| the workflow itself            | UI type               | the folder of tests that runs, the systems, the launch name and its `uiType` attribute |

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
- There is no workflow per UI type and no `uiType` attribute: they run the `daily` / `hourly`
  suite of each provider, a fixed list of tests, not a folder.
- They share one concurrency group without a queue: at most one run waits, and a newer one
  replaces it - a scheduled run that is late is dropped for the next one instead of piling up.
