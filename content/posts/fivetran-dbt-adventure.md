---
title: "Fivetran Dbt Adventure"
date: 2022-06-04T16:26:48-08:00
draft: true
---

## A real adventure

Recently I've been working on converting our custom (and costly) ETL system into a more standardized (and cost effective) ELT system using Fivetran. Fivetran seems great (https://www.fivetran.com) for ingesting variable amounts of data on regular intervals. I'm using it right now to ingest Quickbooks, Shopify, Klaviyo and Plaid data into our Snowflake data warehouse. A problem that I did not envision was Fivetran requiring every connector (something that pulls from a Source and writes to a Destination) to require its own database schema in Snowflake. As in, because we have both a regular Quickbooks connector and a _custom_ Quickbooks connector (because the out of the box connector _doesn't_ ingest Quickbooks Reports), it will write them each to their own schema. For most people this is fine, but my goal is to reduce complexity as much as I can downstream, which means having to keep track of many schemas even for simple queries just doesn't cut it.

Enter Dbt.

## A wild, wild time

Dbt stands for `Data Build Tool`. I don't know the history, but I sense a callback to Sbt, or `Scala Build Tool`. It's not clear to me how they're related but I didn't let that stop me. As I said, we're trying to take many schemas and many tables and consolidate them into a single schema and many tables. So, an example:

```
quickbooks_2980823978490234.invoices => quickbooks.invoices
quickbooks_8293923732987329.invoices => quickbooks.invoices
```

Dbt works quite well for this style of work because each transformation is written to the database as a View, which means I don't have to worry about stale data.

### Problem 1: Relevant schemas
The first problem I could identify was figuring out what schemas in the warehouse even qualified for consolidation. An easy problem, but it was kind of the keystone behind all the upcoming problems. After playing around with some queries to test, I ended up making a table and a macro to solve this problem for me.

`quickbooks.schema_tables`:

```
SELECT table_schema
       , table_name 
FROM information_schema.tables 
WHERE table_schema like 'quickbooks_%' 
AND table_schema not like '%reports%'
AND table_schema not like '%inactive%'
```

`get_schemas_for_table_type`:

```
{% macro get_schemas_for_table_type(table_type) %}
    select table_schema as schema_name from quickbooks.schema_tables where table_name = '{{table_type}}'
{% endmacro %}
```

Now, if I wanted to get relevant schemas for the `accounts` table type, I could call `get_schemas_for_table_type(accounts)` from a Dbt model and have all I needed.

### Problem 2: Columns

Fivetran is great about schemas, in that it maintains and enforces schemas out of the box for official Fivetran connectors, but a real painpoint is that, if data doesn't exist for a certain field in a Fivetran source, it won't write empty columns to your destination (until data does start existing in that field). What this means is that you cannot simply `UNION ALL` across all qualifying tables, because some tables won't necessarily have all the same columns as the others (even if they're technically the same table type and schema).

To solve this, I (yet again) made a Dbt macro to grab shared columns for a table type.

```
{% macro get_columns_for_table(table_name) %}
    {% set schemas_list_proxy = run_query(get_schemas_for_table_type(table_name)) %}
    {% if execute %}
        {% set schemas_list = schemas_list_proxy.columns[0].values() %}
    {% else %}
        {% set schemas_list = [] %}
    {% endif %}

    {% for schem in schemas_list %}
        {% if loop.first %} WITH {% endif %}
        {{schem}} as (
            SELECT column_name
            FROM information_schema.columns
            WHERE table_schema = '{{schem}}'
            AND table_name = '{{table_name}}'
            AND data_type NOT IN ('json', 'jsonb')
        )
        {% if not loop.last %},{% endif %}
    {% endfor %}
    {% for schem in schemas_list %}
        SELECT column_name
        FROM {{schem}}
        {% if not loop.last %} INTERSECT {% endif %}
    {% endfor %}
{% endmacro %}
```

Essentially, this block is creating a list of relevant schemas, and then grabbing all common columns from each of those schemas.


### Putting it together

This part's kind of messy, but:

```{% macro consolidate_schemas(table_name) %}
    {% set cols_list_proxy = run_query(get_columns_for_table(table_name)) %}
    {% if execute %}
        {% set cols_list = cols_list_proxy.columns[0].values() %}
    {% else %}
        {% set cols_list = [] %}
    {% endif %}

    {% set schemas_list_proxy = run_query(get_schemas_for_table_type(table_name)) %}
    {% if execute %}
        {% set schemas_list = schemas_list_proxy.columns[0].values() %}
    {% else %}
        {% set schemas_list = [] %}
    {% endif %}

    with connection_info as (
        SELECT company_id, connection_id, id_slug
        FROM fivetran_schemas.connections
        WHERE institution_type = 'QUICKBOOKS'
    ),

    {% for schema_name in schemas_list %}
        {{schema_name}} AS (
            select 
                ci.company_id,
                ci.connection_id,
                {{company_identifier(schema_name)}} as id_slug,
            {% for col in cols_list %}
                {{col}} {% if not loop.last %},{% endif %}
            {% endfor %}
            from {{schema_name}}.{{table_name}}
            inner join connection_info ci
            on ci.id_slug = id_slug
            where id_slug = {{company_identifier(schema_name)}}
        )
        {% if not loop.last %} , {% endif %}
    {% endfor %}
    {% for schema_name in schemas_list %}
        select *
        from {{schema_name}}
        {% if not loop.last %} union {% endif %}
    {% endfor %}
{% endmacro %}
```

Yet another macro that, when supplied a table type, gets all relevant schemas + all revenant columns and then does a massive `UNION` query against that data to create one big View, but with customer ID as a field in the resultset.
This, then, satisfies my original goal of merging all the `invoice` (this type just as an example) tables into a single `quickbooks.invoice` table.

To actually invoke the macro I wrote a simple script that grabbed every table type from the warehouse and wrote out an individual Dbt model for that type, into the `models` directory of my Dbt app. So then, I ended up with ~60 files that look like this:

```
{{ consolidate_schemas('account') }}
```

Pretty simple!

## Closing thoughts

This solution has some issues. If a new company gets added, the Dbt models must be recompiled to include that company's data in the view. Also, if an existing company gets a new field (because data got updated), we must also rerun the View. To solve this problem I run Dbt as a callback in a bunch of areas of our application, like company registration as an example, to be sure new companies get added with haste.

Overall I enjoyed working with Dbt and will continue to do so!