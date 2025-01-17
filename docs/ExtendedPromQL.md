# MetricsQL

VictoriaMetrics implements MetricsQL - query language inspired by [PromQL](https://prometheus.io/docs/prometheus/latest/querying/basics/).
It is backwards compatible with PromQL, so Grafana dashboards backed by Prometheus datasource should work the same after switching from Prometheus to VictoriaMetrics.

The following functionality is implemented differently in MetricsQL comparing to PromQL in order to improve user experience:
* MetricsQL takes into account the previous point before the window in square brackets for range functions such as `rate` and `increase`.
  It also doesn't extrapolate range function results. This addresses [this issue from Prometheus](https://github.com/prometheus/prometheus/issues/3746).
* MetricsQL returns the expected non-empty responses for requests with `step` values smaller than scrape interval. This addresses [this issue from Grafana](https://github.com/grafana/grafana/issues/11451).
* MetricsQL treats `scalar` type the same as `instant vector` without labels, since users usually don't feel the difference between these types.
  See [the corresponding Prometheus docs](https://prometheus.io/docs/prometheus/latest/querying/basics/#expression-language-data-types) for details.

Other PromQL functionality should work the same in MetricsQL. [File an issue](https://github.com/VictoriaMetrics/VictoriaMetrics/issues)
if you notice discrepancies between PromQL and MetricsQL results other than mentioned above.

MetricsQL provides additional functionality mentioned below, which is aimed towards solving practical cases.
Feel free [filing a feature request](https://github.com/VictoriaMetrics/VictoriaMetrics/issues) if you think MetricsQL misses certain useful functionality.

*Note that the functionality mentioned below doesn't work in PromQL, so it is impossible switching back to Prometheus after you start using it.*

This functionality can be tried at [an editable Grafana dashboard](http://play-grafana.victoriametrics.com:3000/d/4ome8yJmz/node-exporter-on-victoriametrics-demo).

- [`WITH` templates](https://play.victoriametrics.com/promql/expand-with-exprs). This feature simplifies writing and managing complex queries. Go to [`WITH` templates playground](https://victoriametrics.com/promql/expand-with-exprs) and try it.
- Metric names and metric labels may contain escaped chars. For instance, `foo\-bar{baz\=aa="b"}` is valid expression. It returns time series with name `foo-bar` containing label `baz=aa` with value `b`. Additionally, `\xXX` escape sequence is supported, where `XX` is hexadecimal representation of escaped char.
- `offset`, range duration and step value for range vector may refer to the current step aka `$__interval` value from Grafana.
  For instance, `rate(metric[10i] offset 5i)` would return per-second rate over a range covering 10 previous steps with the offset of 5 steps.
- `offset` may be put anywere in the query. For instance, `sum(foo) offset 24h`.
- `offset` may be negative. For example, `q offset -1h`.
- `default` binary operator. `q1 default q2` substitutes `NaN` values from `q1` with the corresponding values from `q2`.
- `histogram_quantile` accepts optional third arg - `boundsLabel`. In this case it returns `lower` and `upper` bounds for the estimated percentile. See [this issue for details](https://github.com/prometheus/prometheus/issues/5706).
- `if` binary operator. `q1 if q2` removes values from `q1` for `NaN` values from `q2`.
- `ifnot` binary operator. `q1 ifnot q2` removes values from `q1` for non-`NaN` values from `q2`.
- Trailing commas on all the lists are allowed - label filters, function args and with expressions. For instance, the following queries are valid: `m{foo="bar",}`, `f(a, b,)`, `WITH (x=y,) x`. This simplifies maintenance of multi-line queries.
- String literals may be concatenated. This is useful with `WITH` templates: `WITH (commonPrefix="long_metric_prefix_") {__name__=commonPrefix+"suffix1"} / {__name__=commonPrefix+"suffix2"}`.
- Range duration in functions such as [rate](https://prometheus.io/docs/prometheus/latest/querying/functions/#rate()) may be omitted. VictoriaMetrics automatically selects range duration depending on the current step used for building the graph. For instance, the following query is valid in VictoriaMetrics: `rate(node_network_receive_bytes_total)`.
- [Range duration](https://prometheus.io/docs/prometheus/latest/querying/basics/#range-vector-selectors) and [offset](https://prometheus.io/docs/prometheus/latest/querying/basics/#offset-modifier) may be fractional. For instance, `rate(node_network_receive_bytes_total[1.5m] offset 0.5d)`.
- Comments starting with `#` and ending with newline. For instance, `up # this is a comment for 'up' metric`.
- Rollup functions - `rollup(m[d])`, `rollup_rate(m[d])`, `rollup_deriv(m[d])`, `rollup_increase(m[d])`, `rollup_delta(m[d])` - return `min`, `max` and `avg`
  values for all the `m` data points over `d` duration.
- `rollup_candlestick(m[d])` - returns `open`, `close`, `low` and `high` values (OHLC) for all the `m` data points over `d` duration. This function is useful for financial applications.
- `union(q1, ... qN)` function for building multiple graphs for `q1`, ... `qN` subqueries with a single query. The `union` function name may be skipped -
  the following queries are equivalent: `union(q1, q2)` and `(q1, q2)`.
- `ru(freeResources, maxResources)` function for returning resource utilization percentage in the range `0% - 100%`. For instance, `ru(node_memory_MemFree_bytes, node_memory_MemTotal_bytes)` returns memory utilization over [node_exporter](https://github.com/prometheus/node_exporter) metrics.
- `ttf(slowlyChangingFreeResources)` function for returning the time in seconds when the given `slowlyChangingFreeResources` expression reaches zero. For instance, `ttf(node_filesystem_avail_byte)` returns the time to storage space exhaustion. This function may be useful for capacity planning.
- Functions for label manipulation:
  - `alias(q, name)` for setting metric name across all the time series `q`.
  - `label_set(q, label1, value1, ... labelN, valueN)` for setting the given values for the given labels on `q`.
  - `label_del(q, label1, ... labelN)` for deleting the given labels from `q`.
  - `label_keep(q, label1, ... labelN)` for deleting all the labels except the given labels from `q`.
  - `label_copy(q, src_label1, dst_label1, ... src_labelN, dst_labelN)` for copying label values from `src_*` to `dst_*`.
  - `label_move(q, src_label1, dst_label1, ... src_labelN, dst_labelN)` for moving label values from `src_*` to `dst_*`.
  - `label_transform(q, label, regexp, replacement)` for replacing all the `regexp` occurences with `replacement` in the `label` values from `q`.
  - `label_value(q, label)` - returns numeric values for the given `label` from `q`.
- `step()` function for returning the step in seconds used in the query.
- `start()` and `end()` functions for returning the start and end timestamps of the `[start ... end]` range used in the query.
- `integrate(m[d])` for returning integral over the given duration `d` for the given metric `m`.
- `ideriv(m)` - for calculating `instant` derivative for `m`.
- `deriv_fast(m[d])` - for calculating `fast` derivative for `m` based on the first and the last points from duration `d`.
- `running_` functions - `running_sum`, `running_min`, `running_max`, `running_avg` - for calculating [running values](https://en.wikipedia.org/wiki/Running_total) on the selected time range.
- `range_` functions - `range_sum`, `range_min`, `range_max`, `range_avg`, `range_first`, `range_last`, `range_median`, `range_quantile` - for calculating global value over the selected time range.
- `smooth_exponential(q, sf)` - smooths `q` using [exponential moving average](https://en.wikipedia.org/wiki/Moving_average#Exponential_moving_average) with the given smooth factor `sf`.
- `remove_resets(q)` - removes counter resets from `q`.
- `lag(q[d])` - returns lag between the current timestamp and the timestamp from the previous data point in `q` over `d`.
- `lifetime(q[d])` - returns lifetime of `q` over `d` in seconds. It is expected that `d` exceeds the lifetime of `q`.
- `scrape_interval(q[d])` - returns the average interval in seconds between data points of `q` over `d` aka `scrape interval`.
- Trigonometric functions - `sin(q)`, `cos(q)`, `asin(q)`, `acos(q)` and `pi()`.
- `median_over_time(m[d])` - calculates median values for `m` over `d` time window. Shorthand to `quantile_over_time(0.5, m[d])`.
- `median(q)` - median aggregate. Shorthand to `quantile(0.5, q)`.
- `limitk(k, q)` - limits the number of time series returned from `q` to `k`.
- `keep_last_value(q)` - fills missing data (gaps) in `q` with the previous value.
- `distinct_over_time(m[d])` - returns distinct number of values for `m` data points over `d` duration.
- `distinct(q)` - returns a time series with the number of unique values for each timestamp in `q`.
- `sum2_over_time(m[d])` - returns sum of squares for all the `m` values over `d` duration.
- `sum2(q)` - returns a time series with sum of square values for each timestamp in `q`.
- `geomean_over_time(m[d])` - returns [geomean](https://en.wikipedia.org/wiki/Geometric_mean) value for all the `m` value over `d` duration.
- `geomean(q)` - returns a time series with [geomean](https://en.wikipedia.org/wiki/Geometric_mean) value for each timestamp in `q`.
- `rand()`, `rand_normal()` and `rand_exponential()` functions - for generating pseudo-random series with even, normal and exponential distribution.
- `increases_over_time(m[d])` and `decreases_over_time(m[d])` - returns the number of `m` increases or decreases over the given duration `d`.
- `prometheus_buckets(q)` - converts [VictoriaMetrics histogram](https://godoc.org/github.com/VictoriaMetrics/metrics#Histogram) buckets to Prometheus buckets with `le` labels.
- `histogram(q)` - calculates aggregate histogram over `q` time series for each point on the graph. See [this article](https://medium.com/@valyala/improving-histogram-usability-for-prometheus-and-grafana-bc7e5df0e350) for more details.
- `topk_*` and `bottomk_*` aggregate functions, which return up to K time series. Note that the standard `topk` function may return more than K time series -
   see [this article](https://www.robustperception.io/graph-top-n-time-series-in-grafana) for details.
   - `topk_min(k, q)` - returns top K time series with the max minimums on the given time range
   - `topk_max(k, q)` - returns top K time series with the max maximums on the given time range
   - `topk_avg(k, q)` - returns top K time series with the max averages on the given time range
   - `topk_median(k, q)` - returns top K time series with the max medians on the given time range
   - `bottomk_min(k, q)` - returns bottom K time series with the min minimums on the given time range
   - `bottomk_max(k, q)` - returns bottom K time series with the min maximums on the given time range
   - `bottomk_avg(k, q)` - returns bottom K time series with the min averages on the given time range
   - `bottomk_median(k, q)` - returns bottom K time series with the min medians on the given time range
