# jiradog

A tool to poll data from JIRA to upload as a metric to Datadog.

## Prerequisites

Dependencies (outside of python standard library)
* JIRA
* Dataog
* pprint
* jinja2

## Installing

Install the .deb file, or build with fpm.

## config.json syntax
An example config file is included:

```
{
  "default": true,
  "local": {
    "log_file": "/var/log/jiradog/jiradog.log",
    "metric_file": "/etc/jiradog/metrics.json"
    },
  "jira": {
    "username": "[username]",
    "password": "[password]",
    "server": "https://example.jira.com"
    },
  "datadog": {
    "app_key": "[app key]",
    "api_key": "[api key]"
    }
}
```

ensure that you replace all words in `[]` and change the `default` tag to `false`, with no quotes (as it is a boolean value).

## metrics.json syntax
(`schema.json` file to be added at a later date.)

How to build a metric.

The information you are pulling from JIRA is built around JIRA's own query language: JQL (JIRA Query Language). Here is an example section from the metrics.json file:

```
  {
    "metric_name": "[name of Datadog metric]",
    "__comment": "[comment]",
    "projects": [
      "KEY1",
      "KEY2",
      "KEY3"
      ],
    "method": "[average|direct]",
    "[numerator|issues]": {
      "source": "jira",
      "jql": "[JQL; check 'JQL with Jinja2 variables' below]",
      "filter": "[jinja if/then statement; must return true]"
      "method": "[ticket_count|mean_time_between_statuses]"
      },
    "denominator": {
      "source": "constant",
      "data": {
        "[KEY1; data applied to project only]": "[data]",
        "[KEY2]": "[data]",
        "[KEY3]": "[data]"
        }
      },
    "grouping": {
      "by": "[sprint]",
      "count": "[number; negative for 'last']",
      "boards:" {
        "KEY1": "[board id]",
        "KEY2": "[board id]",
        "KEY3": "[board id]"
        }
      },
    }
```

Using standard JSON syntax we are defining what information to poll from JIRA, how to process it, and what to name the metric in Datadog.

### script-based filtering
We use a jinja2/nunjucks style filtering with if/then statements. JQL currently doesn't support selecting fixVersions with a "~" (contains) or "LIKE" operator. To get around this, take a look at the `filter` key-value pair in this example data provider block:

```
"filter": "{% if 'GA' in issue.fields.fixVersions[0].name %}true{% endif %}"
```

Following the typical jinja2 if/then statement, we have the statement, which is declared by the surrounding `{%` and `%}`. Inside is a basic structure: `if issue.fields.fixVersions.0.name ~ 'GA'`. `issue.fields.fixVersions.0.name` is the 'path' of the key-value pair in the issue we are looking to compare with the value, in this case `GA`. This will filter all issues to see if they have a `fixVerions` name that contains `GA`.

The jinja statement must return `true`.

### More on the metrics.json

The metrics.json file is a JSON list of dictionaries, each one a 'description'/'assertion' of what is needed out of JIRA, how to process, and what to name the metric in Datadog. Because it is json, be wary of JSON's strict syntax, especially with trailing/missing commas.

### Philosophy

The general philsophy is to use JQL to get the most specific results as possible, within the limitations of JQL. As I expand the script-based filtering, we want to rely on that as little as possible. script-based filtering will be removed if JQL gains the functionality we are looking for. When building a new metric, be as specific as possible with your JQL for issue searches and preferrably use no script-based filtering if possible.

### Style guide

#### JQL

None of the following styles, whether followed or not, will break script functionality.

##### Whitespace
Here is a correctly styled JQL query.

```
project={{project}} AND issuetype=Bug AND status NOT IN (Done,Resolved,Closed)
```

The 'official' whitespace rules are:

- Whitespace = SPACE (`U+0020`)
- No _leading_ or _trailing_ whitespace
- One whitespace character maximum concurrently (there should be no double spaces)
- In paranthesized groups, no whitespace following comma: `(Done,Resolved,Closed)`

##### Fields

When referencing a field in JQL, use camelCase:

- `project`
- `issueType`
- `fixVersions`
- `resolved`
- `updatedDate`

##### Operators

Operators are used to join several conditions or expand on a condition.

- Operators must be written in UPPERCASE (`AND`, `WAS`, `OR`, `NOT`, `IN`, etc)
- Operators should be surrounded by whitespace (`x AND y`, `status NOT IN (foo,bar)`)
- Mathmatical operators (=, !=, >, <, >=, <=, etc) should not be surrounded by whitespace (`x=4`, `assignee!=currentUser()`, `updated>=endOfDay(-90d)`)

##### JQL with Jinja2 variables

- {{project}}
- {{sprint_id}}
- {{sprint_end_date}}

#### Jinja2

None of the following styles, whether followed or not, will break script functionality.

##### Spacing

Lifted from the jinja2 style guide from the chromium project: https://www.chromium.org/developers/jinja#TOC-Spacing

```
  {{foo}}
  {{foo | filter}}
  {% for i in x|filter %}
```

> I.e., no spacing within {{}}, but do space around |, except in block control statements. This is the opposite of Jinja spec convention, which is {{ foo }} and {{ foo|filter }} – reasoning is that {{}} are functioning like parentheses or quotes, hence no spacing inside them; while | is functioning like a binary operator, like arithmetic operations, hence spaces around it. However, in block control statements | has higher precedence than the control flow keywords, and thus omitting spaces makes the precedence clearer.
More pragmatically, {{}} is very widely used for variable substitution, hence want to keep it short and looking like a single unit, while usage of filters via | is logically complex, and thus should stick out visually, with the steps in the pipeline clearly separated. However, adding spaces within a block control statement makes it confusing, due to the spaces around the control flow keywords, and thus spaces around | should be omitted. (von Barth, "Jinja")

##### Comments

> If a block is long, particularly if it is nested, please add a comment at the end to help clarify which block is ending; use the same content as the if condition or for list. (von Barth, "Jinja")

Example comment:

```
{# foo #}
```

### Naming Conventions

#### Datadog Metric Name

This is the jiradog way to name metrics for Datadog, using a similar psuedo-heirarchical dot list that the Datadog agent uses.

```
[source].[Variable][Variable][Group By].[method]
```

- source
  - This is the name of the script: `jiradog`
- [Variable]
  - [p1P2P3P4][issueType][filter][groupBy]
    - [p1P2P3P4]
      - priority list, can be any combination of them:
        - e.g. p1, p2, p3, p4, p1P2, p2P3, p1P2P3, etc.
      - If the metric includes all priorities, omit this part.
    - [issueType]:
      - JIRA issue type
      - Typed in camelCase
      - All non alphanumeric characters are removed
    - [filter]
      - This is to specify more about the search.
      - This could be the issue status we are searching on (Reopened) or a label on the issue (Regression).
      - Written in camelCase
    - [groupBy]
      - PerSprint, PerWeek, etc.
- [method]
  - [mean[TimeToClose|Age]|percent|count|leadTime|cycleTime|taktTime]
  - How the metric is post-processed

- e.g.
  - jiradog.bugsOpen.count
  - jiradog.p1P2BugsPerSprint.meanTimeToClose
  - jiradog.bugsReopened.percent

#### Version Numbering

jiradog follows the three major tenents of the [SemVer](https://semver.org/spec/v2.0.0.html) method of software versioning.

> Given a version number MAJOR.MINOR.PATCH, increment the:
>
> - MAJOR version when you make incompatible API changes,
> - MINOR version when you add functionality in a backwards-compatible manner, and
> - PATCH version when you make backwards-compatible bug fixes.
>
> Additional labels for pre-release and build metadata are available as extensions to the MAJOR.MINOR.PATCH format. (Preston-Warner, 2013)

Build metadata MAY be denoted by appending a hyphen and a series of dot separated identifiers immediately following the patch or pre-release version.

Build version follows [RFC-1912 section 2.2](https://tools.ietf.org/html/rfc1912#section-2.2):

> The recommended syntax is YYYYMMDDnn
> (YYYY=year, MM=month, DD=day, nn=revision number.  This won't
> overflow until the year 4294" (Barr, 1996)

Example: `1.23.4-2018033000`:

- Major version `1`
- Minor version `23`
- Patch version `4`
- Build number `2018033000` => `2018` `03` `30` `00`

Please read the full spec at the link above or in the citations.

#### todo.txt

Things todo are kept in the todo.txt file, following the todo.txt syntax found [here](https://github.com/todotxt/todo.txt).

## Citations
Von Barth, N. (n.d.). Jinja. Retrieved November 09, 2017, from
    https://www.chromium.org/developers/jinja#TOC-Spacing
    
Von Barth, N. (n.d.). Jinja. Retrieved November 09, 2017, from
    https://www.chromium.org/developers/jinja#TOC-Comment-end-of-long-blocks

Preston-Warner, T. (2013). Semantic Versioning 2.0.0. Retrieved November 27, 2017, from
    https://semver.org/spec/v2.0.0.html

Barr, D. (1996). _Common DNS Operational and Configuration Errors_. Retrieved on March 30, 2018,
    from https://tools.ietf.org/html/rfc1912#section-2.2

## Authors

* **Bryce McNab** - *Initial work* - 01/12/2018
