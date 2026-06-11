# Running the DocServer test workflow

Reference for the **Puppeteer DocServer test** workflow form (*Actions* → *Run workflow*).
Each field is described with its purpose and effect. Sections follow the order of the form.

The key idea: the **stand** selects the target server (the URL), and the **branch** selects
the test code. They are independent, so the same branch can be run against different stands
without editing any config by hand.

## 1. Use workflow from (`Branch:` selector)

A native GitHub control, not a form input. It selects the branch of the `run-actions`
repository used for the workflow definition, the composite actions (`.github/actions/...`),
and the branch configuration (`.github/branches.json`).

This is the run-infrastructure branch, not the branch containing the tests. Leave it on
`master` unless you are testing changes to the workflows or actions themselves.

## 2. Stand to test (required)

Dropdown of stands: `kim` or `doc-linux`. The stand selects the **target server**. Its URL
is taken from a repository secret:

| Stand       | Server     | Secret         |
| ----------- | ---------- | -------------- |
| `kim`       | release    | `URL_RELEASE`  |
| `doc-linux` | hotfix     | `URL_HOTFIX`   |

The selected URL is written into the OS config (`testOptions.url` in
`config_chrome_*.json`) before the run, so both the tests and the cleanup target that
server.

The stand also sets the **default** `Dep.Tests` branch (used only when the **branch** field
is empty), via `.github/branches.json`:

| Stand       | Default Dep.Tests branch |
| ----------- | ------------------------ |
| `kim`       | `master`                 |
| `doc-linux` | `doc-linux`              |

To run a branch against a different server, leave the stand set to that server and fill in
the **branch** field (section 4).

The secrets `URL_RELEASE` and `URL_HOTFIX` must be defined in the `run-actions` repository
settings.

## 3. Operating system to run on (required)

`all`, `windows`, `linux`, or `macos`. Determines which runners the tests execute on. `all`
runs on Windows, Linux, and macOS in parallel, producing three independent runs and three
reports. A specific value runs on that operating system only.

## 4. Dep.Tests branch override (optional)

The name of a `Dep.Tests` branch to use instead of the stand default.

- Empty: the branch bound to the selected stand is used (see section 2).
- Set (for example `feature/my-tests`): the workflow switches to that branch, ignoring the
  stand default.

The server is still determined by the stand, not by the branch. This is what allows the
same branch to be run against release and hotfix in turn.

## 5. Specific test file or directory (optional)

A path to a single test or folder inside `tests/`.

- Empty: the full test set runs.
- Set (for example `tests/desktop/word/hometab/font`): only that file or folder runs.

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

- Empty: the name is generated automatically (by run type and chunk number).
- Set: the launch is named as entered.

## 10. ReportPortal API key (optional)

The ReportPortal access key.

- Empty: the repository secret `REPORT_KEY` is used (the normal mode).
- Set: overrides the secret for this run.

Leave empty unless a different key is required.

## Typical run

1. **Use workflow from**: `master`.
2. **Stand to test**: the target server (`kim` for release, `doc-linux` for hotfix).
3. **Operating system**: `all`, or a single operating system.
4. Leave the remaining fields at their defaults.
5. **Run workflow**.

To run a specific branch against both servers, set **branch** to your branch and run the
workflow twice — once with `kim`, once with `doc-linux`.

Per-chunk status appears on the *Actions* tab, results appear in ReportPortal, and the
overall outcome is sent to Telegram.

## Required fields

Fields marked with `*` (**Stand to test**, **Operating system**) are required. They have
default values, so pressing *Run workflow* without changing them is sufficient.

## What each selector controls

| Selector                       | Selects               | Controls                                          |
| ------------------------------ | --------------------- | ------------------------------------------------- |
| **Use workflow from** (top)    | `run-actions` branch  | workflow code, actions, `branches.json`           |
| **Stand**                      | target server         | server URL (from secret) and the default branch   |
| **branch** field               | `Dep.Tests` branch    | the test code (overrides the stand default)       |

## Daily and Hourly workflows

The **Daily** and **SaaS Hourly** workflows are structured similarly, with two differences:

- There is no Stand field and no branch override. Several providers are tested at once
  (docspaceEU, docspaceUS, ncoc, ocis). Each provider's branch is set statically in
  `.github/branches.json` (the `providers` section), and each provider's server URL comes
  from its own config, not from the stand secrets.
- They run on a schedule (cron) and can also be started manually via *Run workflow* with
  the same fields (operating system, test path, chunk size, clean, ReportPortal fields).
