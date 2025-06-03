- Feature Name: otel_support
- Start Date: 2025-06-02
- RFC PR: [seismometer/rfcs#0004](https://github.com/epic-open-source/seismometer-rfcs/pull/4)
- Seismometer Issue: [seismometer/#0000](https://github.com/epic-open-source/seismometer/issues/0000)

# Summary
[summary]: #summary

<!-- One paragraph explanation of the proposed change. -->
Seismometer currently exists as an API generating widgets and plots for use in analyzing model quality. Such an approach, while well-suited
to the user who would like to see at a glance how their model is operating, lacks the customizability to be adapted to custom backends or
processing steps. This feature will add support for emitting metrics in the OpenTelemetry standard, which is not only supported by standard
visualization backends like Prometheus or Jaeger but also allows greater flexibility of data processing and sharing.

# Motivation
[motivation]: #motivation

<!-- Why are we doing this? What use cases does it support? What use cases will it NOT support? What is the expected outcome? -->
Some customers have their own data-processing backends for analyzing the quality of their models. Currently, seismometer is set up in
such a way that the visualization and data generation are fused together. Adding support for emitting OpenTelemetry metrics would keep the
existing notebook and plotting capabilities, but also allow the end user to separate metric generation from visualization. Doing it via
OpenTelemetry in particular will give seismometer the ability to interface with whatever other tools -- of which there are many -- follow
this same open-source standard.

# Detailed design
[design]: #design

<!-- Explain the design in enough detail that somebody familiar with the area would understand it and somebody familiar with the code could implement it.
If this is content to support a new type of evaluation, describe what dataset you will use to support its development. -->

## The Core
At its core, the new proposed telemetry emission would lie in a new `OpenTelemetryRecorder` class, which would provide methods to record
data in the form of OpenTelemetry metrics. In the methods which currently run when new metrics are requested (such as
`BinaryClassifierMetricGenerator.calculate_binary_stats`), calls to the new code would be added, which would then invoke methods from the
OpenTelemetry SDK to emit standardized metrics.

More elaborately, `OpenTelemetryRecorder` would possess the following fields and methods:
- `instruments: dict[str, Gauge]`, where `Gauge` is an OpenTelemetry class for recording metrics. The dictionary would be accessed by name,
so for instance `self.instruments["PPV"]` would be the instrument recording all PPV-related metrics.
- `populate_metrics(self, attributes: dict[str, Any], metrics: dict[str, float])`, which uses `self.instruments` to record the metrics
stored in metrics, with `attributes` storing the parameters of the call. For example, if a score threshold of 0.2 resulted in an accuracy
of 0.25 on 70+ patients, we might call `populate_metrics({"score_threshold":0.2, "Age":"70+"}, {"accuracy":0.25})`, which would then
use `self.instruments["accuracy"]` to record the received data.

## The Output System
While it may be expedient to simply write metrics into files or even to the console, because the entire point of OpenTelemetry support is
to allow integrated use of seismometer with other applications we would have more sophisticated output methods.

Luckily, OpenTelemetry provides such capabilities, by means of "exporters" (which emit telemetry) and "collectors" (which accept it for
later use), which can run over web protocols like HTTP/protobuf. As such, each `MetricGenerator` instance will contain a field of type
`MetricExporter` (which includes basic functionality like the subclass of type `ConsoleMetricExporter` as well as the more extensible
`OTLPMetricExporter`) which will determine where OTLP metrics go -- to a file or console for simple use, or to a collector for more
integrated use.

## Brief Example
For instance, to log metrics to standard out (as an initial/debug step), the necessary code might look something like this:
```py
# (... in a class constructor of some sort ...)
    self.recorder = otel.OpenTelemetryRecorder(metrics=['specificity', 'sensitivity'])

# (... in some sort of data generation method, whose results get passed to plotting code ...)
    plot_data = get_all_the_data(parameters) # a dictionary or dataframe with specificity/sensitivity as keys or column headers
    self.recorder.exhaust_metrics(attributes=parameters, metrics=plot_data)
```

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

<!--
- Why is this design the best of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?
-->

One alternative is to instead access the metric-exporting functionality by means of having decorators on each widget which is meant to
export metrics. However, as I will detail some more below in the unresolved questions, it is worthwhile to consider building seismometer
towards a state where visualization and metric exporting are separated, and accessing the metric-emitting functionality where we define
plotting functions would serve only to entangle these functions more.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

<!--
- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this change?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?
-->

One thing we should consider in the future is how separate we want the visualization and data steps to be from one another. This RFC will
add metric-emitting functionality into the code while keeping current plotting capabilities intact, but it would probably be more elegant
to separate seismometer into two steps: an OpenTelemetry-emitting stage which calculates the metrics, and then an OpenTelemetry-accepting
stage which can create plots or interactive widgets in which to explore the resulting data.
