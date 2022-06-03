# Build Queries in Honeycomb

When working with a dataset or environment, the primary control for constructing queries is the Query Builder. Below is an example of the Builder, ready for you to make changes to the query’s clauses and run it over your data by applying those changes.

![Screenshot of Query Builder]()

## The Query Builder

> Lynn: A practitioner nees to learn how to compose a query and read the results.

A query consists of six clauses:

- [**Visualize**](#visualize): display specific statistics across events.
- [**Where**](#where): filter events based on the value of one or more attributes.
- [**Order By**](#order-by): sort the results.
- [**Group By**](#group-by): split events into groups based on the value of an attribute.
- [**Limit**](#limit): specify a limit on the number of results to return.
- [**Having**](#having): filter results based on aggregate criteria.

A query outputs the results in a graph and the data in a table dependent on your query.

A **Visualize** clause draws a [time series](https://en.wikipedia.org/wiki/Time_series) graph that represents the calculated value over time.

![Query with a visualize clause defined]()

Multiple **Visualize** clauses draws multiple graphs, one for each calculation.

![Query with multiple visualize clauses defined]()

A **Group By** clause draws a time series graph with multiple lines, one for each group. The summary table contains a single row for each group.

![Query with group by clauses defined]()

Leave **Visualize** blank returns the raw event data without any summarization.

> Lynn: This section fails to touch on HEATMAP output or the other possible outputs.

## Working With The Query Builder

Each clause in the Query Builder is accessible through mouse clicks and keyboard shortcuts. As you enter keystrokes into each field a contextual menu appears with proposed clauses to autocomplete. Navigate through these clauses with the down arrow key and up arrow key. Select an entry with a click, tab key, or enter key.

Display a list of the supported keyboard shortcuts by pressing the question mark key `?`.

### Example

![Screenshot illustrating editing the Query Builder]()

The **VISUALIZE** clause is set to `COUNT`, the **GROUP BY** set to `hostname`, and **ORDER BY** set ti `COUNT desc`. The **WHERE** clause is selected and the field contains `app.status`. A contextual menu displays proposed clauses clauses to autocomplete.

## Visualize

An [event](getting-started/events-metrics-logs/#structured-events) is a collection of fields that describe the state of a system when it performed a unit of work. The visualization clause performs a calculation across all events or fields of the events.

| Aggregate                        | Description                                 |
| -------------------------------- | ------------------------------------------- |
| `COUNT`                          | A count of all the events                   |
| `COUNT_DISTINCT(<field_name>)`   | A count of all the events unique for a field |
| `SUM(<numeric_field_name>)`	     | Total all the events for a numeric field    |
| `AVG(<numeric_field_name>)`	     | An average of the events for a numeric field |
| `MAX(<numeric_field_name>)`	     | The maximum value for a numeric field              |
| `MIN(<numeric_field_name>)`	     | The minimum value for a numeric field              |
| `PXX(<numeric_field_name>)`	     | The XXth percentile for a numeric field (i.e. `P10`, `P25`, `P50`, `P75`, `P90`, `P95`, and `P99`) |
| `HEATMAP(<numeric_field_name>)`	 | A heatmap of the distribution for a numeric field |
| `CONCURRENCY`	                   | The concurrency of the current query, for tracing-enabled datasets only. Refer to [Using the Concurrency Aggregate](/working-with-your-data/use-advanced-operators/concurrency/) for more information. |
| RATE_XXX(<numeric_field_name>)	 | The difference between subsequent field values after applying the XXX operator. Refer to [Using the Rate Aggregates](//working-with-your-data/use-advanced-operators/rate/) |

Most operations require a single argument. The `<field_name>` argument represents any event field. The `<numeric_field_name>` argument represnts any event field that is a number storeds as a float or integer. Events with an empty field are ignored and not counted in aggregations.

`COUNT` and `CONCURRENCY` take no arguments.

Defining multiple Visualize clauses is common and can be useful, especially when comparing their outputs to each other. For example, it can be useful to see both the `COUNT` and the `P95(duration)` for a set of events to understand whether latency changes follow volume changes.

While most Visualize queries return a line graph, the `HEATMAP` visualization allows you see the distribution of data in a rich and interactive way. Heatmaps also allow you to use [BubbleUp](/working-with-your-data/bubbleup).

### The Summary Table

The Visualize clause returns both a time series graph and a column of values in the result summary table. Note that in the graph, the values shown are aggregated for each interval at the current granularity. Conversely, the values shown in the summary table are calculated across the entire time range for that query.

These results can be a little surprising for some calculations. The `P95(duration_ms)` across the entire time range may not look quite like the `P95` value at any given point of the curve, because there may be spikes and bumps in the underlying data that are hidden in the time intervals.

The summary table for a `HEATMAP` shows a histogram of values of that field across the full time range.

### Visualize: Basic Case

Given an instrumented web server that has generated the follow events:

| timestamp	       | uri     |	status_code	| response_time_ms |
| ---------------- | ------- | ------------	| ---------------- |
| 2016-08-01 07:30 |  /about |  500         |	126              |
| 2016-08-01 07:45 |	/about |	200	        | 57               |
| 2016-08-01 07:57 |	/docs  |	200	        | 82               |
| 2016-08-01 08:03 |	/docs  |	200         |	23               |

You want to visualize:

- the overall `COUNT` to display how many events have been captured
- the `AVG` of the `response_time_ms` field to display the server's average response time
- the `P95` of the `response_time_ms` field to display the response times in the 95th percentile.

A query with these visualization returns three visualizations:

| COUNT  | 	AVG(response_time_ms) | P95(response_time_ms) |
| ------ | ---------------------- | --------------------- |
| 4	     | 72	                    | 119.4                 |

