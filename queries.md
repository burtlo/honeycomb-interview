# Build Queries in Honeycomb

When working with a dataset or environment, the primary control for constructing queries is the Query Builder. Below is an example of the Builder, ready for you to make changes to the query’s clauses and run it over your data by applying those changes.

![Screenshot of Query Builder]()

## The Query Builder

A query in Honeycomb consists of six clauses:

- Visualize: visualize specific statistics across events.
- Where: choose events based on some additional criteria.
- Group By: split events into groups based on the value of some attribute.
- Order By: sort the results.
- Limit: specify a limit on the number of results to return.
- Having: filter results based on aggregate criteria.

The default output for most queries will be a time series and a summary table, though the precise composition will depend on the composition of your query:

- Specifying a Visualize clause will cause a time series to be drawn representing that calculated value over time; multiple Visualize clauses will result in multiple graphs, one for each calculation.
- Specifying a Group By clause will result in the time series drawing multiple lines, one for each group. The summary table will contain a single row for each unique group.
- Leaving Visualize blank will result in raw event data being returned without any summarization.

Let us take a closer look at how to use each of these clauses, and cases in which each of them can be particularly useful. When we discuss the effects of each of these operations, **events** are inputs to the query (the series of raw payloads you sent that match a set of criteria). **Results**, on the other hand, refer to the output of a query after any applicable processing or aggregation.

## Working With The Query Builder

Click on any box in the query builder to edit the clauses there. In this shot, the user has set Visualize to be `COUNT` and has Grouped By `hostname`. Now, they are adding a new Where clause on `app.status_code`. Honeycomb autocomplete helps construct the query.

![Screenshot illustrating editing the Query Builder]()

> Honeycomb supports keyboard shortcuts! See a list of supported actions by typing ? while on the Query Builder page.

## Visualize

Honeycomb supports a wide range of calculations to provide insight into your events. When a grouping is provided, calculations occur within each group; otherwise, anything calculated is done so over all matching events.

For example, say you have collected the following events from your web server logs:

| Timestamp	| uri |	status_code	| response_time_ms |
| 2016-08-01 07:30 |  /about |  500 |	126 |
| 2016-08-01 07:45 |	/about |	200	| 57  |
| 2016-08-01 07:57 |	/docs  |	200	| 82  |
| 2016-08-01 08:03 |	/docs  |	200 |	23  |

Specifying a visualization for a particular attribute (`P95(response_time_ms)`for example) means to apply the aggregation function (in this case, `P95`, or taking the 95th [percentile](https://en.wikipedia.org/wiki/Percentile)) over the values for the attribute (`response_time_ms`) across all input events.

Defining multiple Visualize clauses is common and can be useful, especially when comparing their outputs to each other. For example, it can be useful to see both the `COUNT` and the `P95(duration)` for a set of events to understand whether latency changes follow volume changes.

While most Visualize queries return a line graph, the `HEATMAP` visualization allows you see the distribution of data in a rich and interactive way. Heatmaps also allow you to use [BubbleUp](/working-with-your-data/bubbleup).

## Visualize: Basic Case

Scenario: we want to capture overall statistics for our web server. Given our four-event dataset described above, consider a query which contains:

- Visualize the overall COUNT
- Visualize the AVG of response_time_ms values
- Visualize the P95 of response_time_ms values
These calculations would return statistics across the entirety of our dataset:

| COUNT  | 	AVG(response_time_ms) | P95(response_time_ms)
| 4	 | 72	 | 119.4 |

### Visualize Operations

Most visualize operations take a single argument, with the exception of `COUNT` and `CONCURRENCY`, which take no arguments. Events that do not have a relevant attribute are ignored, and will not be counted in aggregations.

| Aggregate | 	Meaning |
| COUNT | 	The number of events |
| COUNT_DISTINCT(<field_name>) | 	The number of different values |
| SUM(<numeric_field_name>)	 | The sum of the field value |
| AVG(<numeric_field_name>)	 | The average value of the field |
| MAX(<numeric_field_name>)	 | The maximum value of the field |
| MIN(<numeric_field_name>)	 | The minimum value of the field |
| PXX(<numeric_field_name>)	 | The XXth percentile of the field |
| HEATMAP(<numeric_field_name>)	 | A heatmap of the distribution of that field |
| CONCURRENCY	 | The concurrency of the current query, for tracing-enabled datasets only. This is a complex operation; please see the detailed explanatory page on it. |
| RATE_XXX(<numeric_field_name>)	 | The difference between subsequent field values after applying the XXX operator. Learn more about the RATE operators. |

The `<numeric_field_name>` argument refers to a float or integer field.

### The Summary Table

The Visualize clause returns both a time series graph and a column of values in the result summary table. Note that in the graph, the values shown are aggregated for each interval at the current granularity. Conversely, the values shown in the summary table are calculated across the entire time range for that query.

These results can be a little surprising for some calculations. The `P95(duration_ms)` across the entire time range may not look quite like the `P95` value at any given point of the curve, because there may be spikes and bumps in the underlying data that are hidden in the time intervals.

The summary table for a `HEATMAP` shows a histogram of values of that field across the full time range.

## Where

Sometimes you want to constrain the events by some attribute besides time: ignoring an outlier case, for example, or isolating events triggered by a particular actor or circumstance.

For example, say you have collected the following events from your web server logs:

| Timestamp | 	uri	 | status_code |
| 2016-08-01 08:15 | 	/about | 	500 |
| 2016-08-01 08:22 | 	/about | 	200 |
| 2016-08-01 08:27 | 	/docs | 	403 |

You can define any number of arbitrary constraints based on event values. Where clauses work in concert with the specified time range to define the events that are ultimately considered by any Group By or Visualize clauses.

Note that the Where clause does not require string delimiters or escape characters; to match a url of `/docs`, simply enter `url = /docs`.

### Where: Basic Case

Scenario: we want to understand the frequency of unsuccessful web requests. Given our three-event dataset described above, consider a query which contains:

- Visualize the overall COUNT
- Where status_code != 200

The Where clause removes the successful event (our `/about` web request returning a `200`) from consideration, and only counts the first and third events towards our Visualize clause:

| COUNT |
| 2 |

### Where: Multiple Clauses

Scenario: we want to refine our constraints further, to span multiple attributes for each event. Combining where clauses returns events that satisfy either the **intersection** of all specified Where clauses, or the **union**. Given our three-event dataset [described above](#above), consider a query which contains:

- Where uri = "/about"
- AND status_code != 200

As all three events are considered by the Where clauses, only the first one satisfies both:

| Timestamp |	uri	| status_code |
| 2016-08-01 08:55 |	/about | 500 |

Honeycomb also allows you to look at the **union** of clauses by setting to an `OR`.

- Where status_code = 500
- OR status_code = 403

| Timestamp | 	uri	| status_code |
| 2016-08-01 08:55 | 	/about | 	500 |
| 2016-08-01 08:27 | 	/docs | 	403 |

### Where Operations

`WHERE` operations may take one or more attributes. Events are only counted if they have a relevant attribute, except in the case of the `does-not-exist` operation.

| Operation |	Opposite |	Arguments	| Meaning |
| `=` |	`!=` |	1 |	Exact numerical or string match |
| `starts-with` |	`does-not-start-with` |	1	| String start match |
| `exists`	| `does-not-exist` |	0 |	Checks for non-null values |
| `>, >=` | 	`<, <=` |	1 |	numerical comparison |
| `contains` |	`does-not-contain` |	1	| string inclusion: checks whether the attribute matches a substring of the value |
| `in` |	`not-in` |	1+ |	list inclusion: checks whether the attribute matches any item in the list |

The syntax for the `in` operator does not use parentheses: `request_method in GET,POST`

### Group By

Being able to separate a series of events into groups by attribute is a powerful way to compare segments of your dataset against each other.

For example, say you have collected the following events from your web server logs:

| Timestamp |	uri |	status_code |
| 2016-08-01 07:30 |	/about | 500 |
| 2016-08-01 07:45 | 	/about | 200 |
| 2016-08-01 07:57 |	/docs  | 200 |

You might want to analyze your web traffic in groups based on the `uri` ("/about" vs “/docs”) or the `status_code` (500 vs 200). Choosing to group by `uri` would return two result rows: one representing events in which `uri="/about"` and another representing events in which `uri="/docs"`. Each of these grouped results rows will be represented by a single line on a graph.

Grouping by more than one attribute will consider each unique combination of values as a single group. Here, choosing to group by both `uri` and `status_code` will return three groups: `/about`+`500`, `/about`+`200`, and `/docs`+`200`.

Grouping, paired with calculation, can often reveal interesting patterns in your underlying events—grouping by `uri`, for example, and calculating response time stats will show you the slowest (or fastest) `uri`s.

**TIP**: Honeycomb supports grouping your data based on any attribute in an event, though you will likely receive the clearest results by choosing an attribute with an uneven distribution within your data.

The Group By dropdown list also has a shortcut to [create a derived column].

### Grouping and Visualizing: Better Together

Scenario: we want to examine performance of our web server by endpoint. Given our four-event dataset [described above](#above), consider a query which contains:

- Group by `uri`
- Visualize the overall `COUNT`
- Visualize the `AVG` of `response_time_ms` values

Pairing a Grouping clause with a Visualize clause results in events being grouped by `uri`; Honeycomb draws one line for each group, and calculates statistics within each group:

| uri	| COUNT |	AVG(response_time_ms) |
| /about |	2	| 91.5 |
| /docs	| 2 |	52.5 |

This technique is particularly powerful when paired with an Order By and a Limit to return “Top K”-style results.

In this figure, the user has a `VISUALIZE` by `COUNT`, `GROUP BY eventtype`. The two curves, in purple and orange, show the two groups. The popup shows that the user is hovering the `trace_span` eventtype, which has a count of 405 in that 15-second time range.

![Query Builder with two result rows]()

When you move your cursor over the results table at the bottom of the page, each row is highlighted in turn. In the image below, the user has highlighted `request` and sees the orange line highlighted, and the purple line dimmed.

![Query Builder highlighting the request row]()

Rollover for heatmaps is slightly different, as described on the [Heatmaps](/heatmaps) page.

## Order By

Order By clauses define how rows will be sorted in the results table.

For example, say you have collected the following events from your web server logs:

| Timestamp | 	uri | 	status_code | 	response_time_ms |
| 2016-08-01 09:17 | 	/about | 	200 | 	57 |
| 2016-08-01 09:18 |	/about |	500	| 234 |
| 2016-08-01 09:20 |	/404	| 200	| 12 |
| 2016-08-01 09:25 |	/docs	| 200 |	82 |

You can define any number of Order By clauses in a query and they will be respected in the order they are specified.

The Order By clauses available to you for a particular query are influenced by whether any Group By or Visualize clauses are also specified. If none are, you may order by any of the attributes contained in the dataset. However, once a Group By or Visualize clause exists, you may only order by the values generated by those clauses.

### Order By: Basic Case

Scenario: we just want to get a sense of the slowest endpoints in our web server. Given our four-event dataset [described above](#above), consider a query which contains:

- Order by `response_time_ms` in descending (`DESC`) order
- Limit to 1 result

Remember that when no Visualize clauses are defined, we simply return raw events as the result rows:

| Timestamp |	uri |	status_code |	response_time_ms |
| 2016-08-01 09:18 |	/about |	500	| 234 |

### Order By, Paired with Visualize and Group By Clauses

Scenario: we want to capture statistics for our web server and know what we are looking for (long `response_time_ms`s). Given our four-event dataset described above, consider a query which contains:

- Visualize the P95 of `response_time_ms` values
- Group by `uri`
- Order by `P95(response_time_ms)` in descending (`DESC`) order
- Limit to the first `2` results

Our Group By and Visualize queries influence **what** will be returned as result rows (`uri` and the `P95(response_time_ms)` for events within each distinct `uri` group), while the Order by determines the sort order of those results (longest `P95(response_time_ms)` first) and the Limit throws away any results beyond the top `2`:

| uri |	P95(response_time_ms) |
| /about |	225.15 |
| /docs |	82 |

As you can see, any results referencing the event with `uri="/404"` was excluded from our result set as a result of its relatively low `response_time_ms`.

This sort of Top K query is particularly valuable when working with high-cardinality data sets, where a Group by clause might split your dataset into a very large number of groups.

## Limit

The Limit clause provides a maximum number of result rows to return. By default, queries return 100 result rows. The Limit clause allows you to specify up to 1000 rows.

## Having

The Having clause allows you to filter on the results table. This operation is distinct from the Where clause, which filters the underlying events. A Having filter can help further refine your query results when grouping on a high-cardinality attribute, which can result in many different rows in the result table. It can also work in tandem with an Order By clause. Order By allows you to order the results, and Having filters them.

Like Order By, Having selects its series from the results table.

For example, consider a query on Visualize `COUNT`, `P95(duration_ms)`, Group By `endpoint`. This query would show how many times each endpoint ran, and how often it did so. The results table from this query might look something like this:

| endpoint |	COUNT |	P95( duration_ms ) |
| /add-to-cart | 521 |	45 |
| /remove-from-cart |	1021 |	54 |
| /unused-endpoint |	2 |	1500 |
| /empty-page |	10 |	1700 |

You might want to ignore the rarest entries when they are least likely to be useful. One way to do that is to add a Having clause:

The clause `HAVING COUNT > 100` will filter to only results with more than 100 hits on them. You can then `ORDER BY P95( duration_ms )` to sort the results to find the slowest endpoints.

Having works by filtering specifically on aggregations across all results in the selected time range. It currently does not filter time periods that match the criteria. Take a look at the following example. Each event has counts ranging from 0 - 9 across the time range.

![Query Builder showing the difference between total having across all results and ever having one specific result]()

You may want to filter specifically on counts greater than 5 in the example and add the clause `HAVING COUNT > 5`. This will still return all results in the image, since Having filters based off the total in the result table where all values are shown to have counts > 5.

### Having Clause Options

The Having clause always refers to one of the Visualize clauses. It then takes one or more numeric arguments.

| Operator |	Opposite |	Arguments |	Meaning |
| = |	!= |	1 |	numerical | equality |
| >, >= |	<, <= |	1 |	comparison |
| in |	not-in |	1+ |	existence in a list |

The `in` operator compares where a value is one of a set: `COUNT in 10, 20, 30` checks whether the COUNT is precisely one of those values.

## Dataset Switcher

The dataset switcher allows you to change the dataset you are working on without changing the query.

![Screenshot illustrating using the Dataset Switcher]()

When you switch datasets, Honeycomb will load the existing query on that dataset, but it will not run it automatically. You can execute it by typing Shift + Enter or clicking the Run Query button. You can also clear the query by clicking the Clear link underneath the Run Query button.

This functionality is especially useful when switching between testing and production datasets, where much of the schema overlaps. It is also useful when you need to view a specific time range across multiple datasets, as the time range of the query is included when you switch datasets.

If there are fields in the query that do not exist in the new dataset, Honeycomb will display an informational notice letting you know that those fields were removed from the query.

![Screenshot illustrating the informational message]()

> Note: In [Secure Tenancy](/secure_tenancy), queries are not preserved on dataset switching.

Queries Against all Datasets

Honeycomb supports running queries against multiple datasets in your environment. This may be useful if you need to see events that span across multiple services.

## Time Comparison

Honeycomb supports running queries across different periods of time, so you can compare results to see how your systems change.

You can run a Time Comparison query by selecting a time range or toggling “Compare To” above your query results. Honeycomb then will run a query as defined in the Query Builder and another query for the time comparison range selected.

Time comparison queries are currently only in the Query Builder UI and are not supported in the API.