# 🛠️📦 dbt Links 

## dbt Additional Resources
- [dbt Mesh FAQs](https://docs.getdbt.com/best-practices/how-we-mesh/mesh-5-faqs)
- [Awesome dbt, a curated list of dbt resources](https://github.com/Hiflylabs/awesome-dbt#integrations)
- [SODA, tool to find, analyze, and solve data issues](https://docs.soda.io/soda/integrate-dbt.html)
- [Integrating dbt w/ Datadog](https://docs.getdbt.com/guides/serverless-datadog?step=1)

## learn dbt
- [learn.dbt](https://learn.getdbt.com/catalog)
  - [fundamentals]https://learn.getdbt.com/courses/dbt-fundamentals
  - [dbt & snowflake]https://courses.getdbt.com/courses/dbt-cloud-and-snowflake-for-developers
  - [refactoring SQL]https://courses.getdbt.com/courses/refactoring-sql-for-modularity
  - [advanced deployment w/ dbt Cloud]https://courses.getdbt.com/courses/advanced-deployment

## dbt code completion
- [post on setup](https://discourse.getdbt.com/t/how-we-set-up-our-computers-for-working-on-dbt-projects/243)
  ### helper links
  - [ohmyzsh](https://github.com/ohmyzsh/ohmyzsh)
  - [dbt-completion.bash](https://github.com/dbt-labs/dbt-completion.bash)
  - [useful onboarding script](https://gitlab.com/gitlab-data/analytics/blob/master/admin/onboarding_script.zsh)





# cheat sheet

## git workflow for dbt
#### Go to dbt project

```bash
cd ~/projects/{dbt project name}
```

#### Start the pipenv shell

```bash
pipenv shell
```

#### Update master

```bash
git fetch
git pull
```

#### Checkout branch

#### New branch:

```bash
git checkout -b {new_branch name}
```

#### From PR:

```bash
git pull origin pull/{PR Number}/head
```

### Update models locally

```bash
dbt run --models staging_{name of model}
```

#### Add `+` to run all referenced by and/or depends on models

`+model_c` for all upstream models, also runs `model_a` and `model_b`

`model_a+` for all downstream models, also runs `model_b` and `model_c`

`+model_b+` for all up & down stream models, also runs `model_a` and `model_c`

#### Make sure SQL syntax is correct

```bash
sqlfluff fix -f -n {full path of file}
```


#### Update/Create model

```bash
dbt run —models {model name}
dbt test —models {model name}
```

#### Update schema.yml & View local doc changes

```bash
dbt docs generate && dbt docs serve --port 8001
```

#### Run GIT commands

```bash
git add {files}
git commit -m "message here"
git push --set-upstream origin {branch here}
```

#### Either wait for PR review or force merge 🫣 in emergency

```bash
!merge
```

#### Switch back to master, update and delete branch

```bash
git checkout master
git fetch
git merge
git branch -d {local_branch}
```

## More notes

### Update Incremental models

#### update models as needed and run the following locally 

```bash
dbt run -s {model name} --full-refresh
```

#### if updating more than one file and you want to run dbt for all of them

```bash
dbt run -s {model name} {another model name} path_to/models/* --full-refresh
```

#### Once tested and merged into production (make sure buildkite successfully pushed changes), you will need to trigger a full refresh in dagster

1. go to dagster
2. open `ad-hoc` either by searching or navigating to it.
3. Go into `Launchpad` tab
4. Under `Preset` select “Run” and under `Mode` select “Prod”. Dagter will give a template to run ad-hoc models.
5. Replace `model_name` with necessary models to update, `-` for each model.
6. At the same level is `models:` & `task_tags:` add `full-refresh: True`
7. When ready, hit `Launch Run` in the bottom right corner

### Update Seed files

#### Update all seed files

```bash
dbt seed
```

#### Update individual seed

```bash
dbt seed --select {seed name}
```

#### Update individual seed and rebuild table structure

```bash
dbt seed --select {seed name} --full-refresh
```

### Building a snapshot via template

replace `{ }`
```sql
-- template for building a snapshot model

select 

		'{% snapshot {source domain name}_' || lower(TABLE_NAME) || '_snapshot %}' || chr(10) || chr(10) ||
		'{{' || chr(10) ||
		 '  config(' || chr(10) ||
		  '    unique_key=''' || 'id' || ''',' || chr(10) ||
		    '    strategy=''timestamp'',' || chr(10) ||
		    '    updated_at=''' || lower(column_name) || '::timestamp_ntz'',' || chr(10) ||
		    '  )' || chr(10) ||
		'}}' || chr(10) ||
		chr(10) ||
		'select *' || chr(10) ||
		'from {{ source(''catalog_api'', ''' || lower(TABLE_NAME) || ''') }}' || chr(10) ||
		'where ' || '_sdc_batched_at' || ' >= {{ safe_max_timestamp(this, ''' || '_sdc_batched_at' || ''') }}' || chr(10)

from information_schema.columns
where table_schema = '{source domain schema}'
and table_catalog = '{source domain catalog}'
and (column_name ilike '%modif%' or column_name ilike '%update%')
and data_type = 'TIMESTAMP_TZ'
```

### Building a staging model from snapshots

replace `{ }`

```sql
-- template for building base staging models from snapshots

select 

'{{' || char(10)
|| '  config(' || char(10)
|| '    materialized=''table'',' || char(10)
|| '    unique_key=''id'',' || char(10)
|| '  )' || char(10)
|| '}}' || char(10)
|| char(10)
|| 'with stitch_fixed as (' || char(10)
|| '  {{ stitch_utils.coalesce_fields(source(''{snapshot domain}_snapshots'', ''' || lower(table_name)  || ''')) }}' || char(10)
|| '  where dbt_valid_to is null' || char(10)
|| ')' || char(10)
|| 'select' || char(10)
|| ' *' || char(10)
|| 'from stitch_fixed' || char(10),

    'staging_' || replace(lower(table_name), '_snapshot', '') || '.sql' as file_name,
    'select * from {analyst''s schema}.{source domain}.staging_' || replace(lower(table_name), '_snapshot', ''),
    '  - name: ' || lower(table_name),
    *
from information_schema.tables
where 1=1
and table_schema = '{source domain}'
and table_catalog = '{domain snapshot catalog}'
```
