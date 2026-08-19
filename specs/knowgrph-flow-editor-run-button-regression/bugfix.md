# Bugfix Requirements Document

## Introduction

The Knowgrph Flow Editor's "Run" button on both `source_input` (InputWidget) and `compute_summary` (ComputeWidget) widget cards has regressed: clicking "Run" now triggers a `*.json` file download instead of the intended action. The downloaded file contains serialized node or graph frontmatter data, indicating the click event is either propagating to a parent `<a download>` element with a JSON blob `href`, or the action dispatcher is generating a Blob URL from the node/frontmatter payload and triggering a browser download rather than calling the intended handler. On `source_input`, the "Run" button should dispatch a `runDownstream` trigger targeting `compute_summary`. On `compute_summary`, it should dispatch a `compute` trigger. This regression broke a previously working flow execution path, blocking users from running their computing flows from within the canvas.

## Bug Analysis

### Current Behavior (Defect)

1.1 WHEN the user clicks the "Run" button (id: "run", trigger: "runDownstream") on the `source_input` InputWidget card THEN the system triggers a `*.json` file download containing serialized node or graph frontmatter data instead of dispatching the `runDownstream` action to `compute_summary`

1.2 WHEN the user clicks the "Run" button (id: "run", trigger: "compute") on the `compute_summary` ComputeWidget card THEN the system triggers a `*.json` file download containing serialized node or graph frontmatter data instead of dispatching the `compute` action on the widget

1.3 WHEN any widget card action button with a non-download trigger is clicked THEN the system incorrectly produces a `*.json` file download — caused by either the click event propagating to a parent `<a download>` element with a JSON blob `href`, or the action dispatcher serializing the node/frontmatter payload into a Blob URL and invoking the browser's native download handler instead of the intended action handler

### Expected Behavior (Correct)

2.1 WHEN the user clicks the "Run" button on the `source_input` InputWidget card THEN the system SHALL dispatch the `runDownstream` trigger with targets `["compute_summary"]` without initiating a file download

2.2 WHEN the user clicks the "Run" button on the `compute_summary` ComputeWidget card THEN the system SHALL dispatch the `compute` trigger on that widget without initiating a file download

2.3 WHEN a widget card action button is clicked THEN the system SHALL call `event.preventDefault()` and `event.stopPropagation()` on the click event before resolving the action, and SHALL execute only the handler corresponding to the button's `trigger` field value, ensuring no `*.json` file download or other default browser behavior is initiated

### Unchanged Behavior (Regression Prevention)

3.1 WHEN the user clicks the "Edit" button (trigger: "openFieldEditor") on the `source_input` widget card THEN the system SHALL CONTINUE TO open the field editor for `input_query` without regression

3.2 WHEN the user clicks the "Reset" button (trigger: "clearOutputs") on the `compute_summary` widget card THEN the system SHALL CONTINUE TO clear the output fields (`output`, `imageUrl`, `outputSrcDoc`) without regression

3.3 WHEN the `runDownstream` trigger is dispatched to `compute_summary` THEN the system SHALL CONTINUE TO invoke the compute function and update the widget's output fields as defined in `canvas:runAction`

3.4 WHEN the `compute` trigger completes successfully on `compute_summary` THEN the system SHALL CONTINUE TO update `run_status`, `template_flow_demo.active_graph_mutated`, and `template_flow_demo.run_id` side effects as specified

3.5 WHEN the flow editor renders widget card action buttons THEN the system SHALL CONTINUE TO display all buttons (`run`, `edit`, `reset`) with their correct labels, icons, and primary styling
