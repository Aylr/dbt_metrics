# dbt_metrics

## About
This dbt package generates queries based on [metrics](https://docs.getdbt.com/docs/building-a-dbt-project/metrics), introduced to dbt Core in v1.0. 

## :warning: A note on `ref`s
To enable the dynamic referencing of models necessary for macro queries through the dbt Server, queries generated by this package do not participate in the DAG and `ref`'d nodes will not necessarily be built before they are accessed. Refer to the docs on [forcing dependencies](https://docs.getdbt.com/reference/dbt-jinja-functions/ref#forcing-dependencies) for more details.

## Usage
Access metrics like any other macro: 
```
select * 
from {{ metrics.metric(
    metric_name='new_customers',
    grain='week',
    dimensions=['plan', 'country'],
    secondary_calcs=[
        {
            "type": "period_to_date", 
            "aggregate": "sum", 
            "period": "year", 
            "alias": "ytd_sum"
        },
        {
            "type": "period_over_period",
            "lag": 1,
            "how": "ratio",
        },
        {
            "type": "rolling",
            "window": 3,
            "aggregate": "average"
        }
    ]
) }}
```

## Secondary calculations
_Documentation tk once the terminology has stabilised_

## Customisation
Most behaviour in the package can be overridden or customised.

### Calendar 
The package comes with a basic calendar table, running between 2010-01-01 and 2029-12-31 inclusive. You can replace it with any custom calendar table which meets the following requirements:
- non-ephemeral (i.e. materialized as a table or view)
- contains the following columns: `date_day`, `date_week`, `date_month`, `date_quarter`, `date_year`. 

To do this, set the value of the `dbt_metrics_calendar_model` variable in your `dbt_project.yml` file: 
```
config-version: 2
[...]
vars:
    dbt_metrics_calendar_model: ref('my_custom_table')
```

### Time Grains 
The package protects against nonsensical secondary calculations, such as a month-to-date aggregate of a data which has been rolled up to the quarter. If you customise your calendar (for example by adding a [4-5-4 retail calendar](https://nrf.com/resources/4-5-4-calendar) month), you will need to override the `get_grain_order()` macro. In that case, you might remove `month` and replace it with `month_4_5_4`. All date columns must be prefixed with `date_` in the table, but this is not necessary in the model config.

### Custom aggregations 
To create a custom primary aggregation (as exposed through the `type` config of a metric), create a macro of the form `metric_my_aggregate(expression)`, then override the `aggregate_primary_metric(aggregate, expression)` macro to add it to the dispatch list. The package also protects against nonsensical secondary calculations such as an average of an average; you will need to override the `get_metric_allowlist()` macro to both add your new aggregate to to the existing aggregations' allowlists, and to make an allowlist for your new aggregation:
```
    {% do return ({
        "average": ['max', 'min'],
        "count": ['max', 'min', 'average', 'my_new_aggregate'],
        [...]
        "my_new_aggregate": ['max', 'min', 'sum', 'average', 'my_new_aggregate']
    }) %}
```

To create a custom secondary aggregation (as exposed through the `secondary_calcs` parameter in the `metric` macro), create a macro of the form `metric_secondary_calculations_my_calculation(metric_name, dims, config)`, then override the `metric_secondary_calculations(metric_name, dims, config)` macro to add it to the dispatch list. 

### Secondary calculation column aliases
Aliases can be passed into the `calcs` object. If no alias is provided, one will be automatically generated. To modify the existing alias logic, or add support for a custom secondary calculation, override `secondary_calculation_alias(calc_config, grain)`.

## 🧪 Experimental behaviour 
:warning: This behaviour is subject to change in future versions of dbt Core and this package.

### Dimensions on calendar tables
You may want to aggregate metrics by a dimension in your custom calendar table, for example `is_weekend`. _In addition to_ the primary `dimensions` list, add the following `meta` properties to your metric, with the model's dimensions first and the calendar's dimensions second:
```
      meta: 
        dimensions: 
          - type: model
            columns:
              - plan
              - country
          - type: calendar
            columns: 
              - is_weekend
```
You can then access the additional dimensions as normal:
```
select * 
from {{ metrics.metric(
    metric_name='new_customers',
    grain='week',
    dimensions=['plan', 'country', 'is_weekend'],
    secondary_calcs=[]
) }}
```