# `gha-buildevents` Action

[![Integration][ci-integration-badge]][ci-integration-link]

[ci-integration-badge]: https://github.com/kvrhdn/gha-buildevents/workflows/Integration/badge.svg
[ci-integration-link]: https://github.com/kvrhdn/gha-buildevents/actions?query=workflow%3AIntegration

This GitHub Action instruments your workflows using [Honeycomb's buildevents tool][buildevents]. It populates the trace with metadata from the workflow and will always send a trace for the build, even if the build failed.

`gha-buildevents` has to be added to every job that should be instrumented. Every job will create a separate trace.

⚠️ Limitations:

- this action currently only works on Linux hosts (though technically support could be added for other platforms)
- if downloading or executing buildevents fails, the entire job will fail

[buildevents]: https://github.com/honeycombio/buildevents

## How to use it

Put the action at the start of your workflow:

```yaml
- uses: kvrhdn/gha-buildevents@v1
  with:
    # Required: a Honeycomb API key - needed to send traces.
    apikey: ${{ secrets.BUILDEVENTS_APIKEY }}

    # Required: the Honeycomb dataset to send traces to.
    dataset: gha-buildevents_integration

    # Required: the job status, this will be used in the post section and sent
    # as status of the trace. Must always be ${{ job.status }}.
    job-status: ${{ job.status }}

    # Optional: this should only be used in combination with matrix builds. Set
    # this to a value uniquely describing each matrix configuration.
    matrix-key: ${{ matrix.value }}

# ... the rest of your job ...

# 'buildevents build' will automatically run as a post action.
```

`gha-buildevents` is a _wrapping action_. This means it has a post section which will run at the end of the build, after all other steps. In this final step the trace is finalized using `buildevents build`. Since this step runs always, even if the job failed, you don't have to worry about traces not being sent.

### Inputs

Name         | Required | Description                                          | Type   | Default
-------------|----------|------------------------------------------------------|--------|--------
`apikey`     | yes      | API key used to communicate with the Honeycomb API.  | string |
`dataset`    | yes      | Honeycomb dataset to use.                            | string |
`job-status` | yes      | The job status, must be set to `${{ job.status }}`.  | string |
`matrix-key` | no       | Set this to a key unique for this matrix cell.       | string |

### Outputs

No outputs are set, but the following environment variables are set:

- `TRACE_ID`: the trace ID, this defaults to the run number and should be used in all invocation of `buildevents`

## Good to know

### Trace IDs

Trace IDs follow a fixed pattern:

```
<owner>/<repo>-<workflow>-<job>-<run number>-<random number>
```

For example, a workflow run in this repository might be called `kvrhdn/gha-buildevents-Integration-smoke-test-20144-1738717406`. 

This can be convenient when filtering traces, for instance to show all traces related to the the workflow `Integration` you could add a filter in the Honeycomb UI: `trace.trace_id contains Integration`.

## Example

This repository has its own workflow which will run every hour. See [.github/workflows/integration.yaml](./.github/workflows/integration.yaml).

This workflow will create the following trace in Honeycomb:

![Trace created in Honeycomb](./example-trace.png)

## Adding additional spans

After `gha-buildevents` has run, `buildevents` will be available on the path. You can use the `buildevents` executable to add additional spans.

`gha-buildevents` sets an environment varible `TRACE_ID`. The trace ID should be used with all buildevents commands to ensure the trace is continued.

To learn more about buildevents and how to use it, checkout [honeycombio/buildevents][buildevents].

```yaml
  # Record the start of the step and, for conveniance, set the step ID that will
  # be used for all commands.
- run: |
    echo ::set-env name=STEP_ID::0
    echo ::set-env name=STEP_START::$(date +%s)

  # Wrap the commands that should be traced with 'buildevents cmd'
- run: |
    buildevents cmd $TRACE_ID $STEP_ID sleep-5 -- sleep 5

  # Wrap up the step
- run: |
    buildevents step $TRACE_ID $STEP_ID $STEP_START 'step 1'
```

## License

This Action is distributed under the terms of the MIT license, see [LICENSE](./LICENSE) for details.
