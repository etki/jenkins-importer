# Huge draft warning

This project exists only as idea yet. However, i most certainly will 
implement it soon, because managing jenkins is still a PITA.

# Jenkins Importer

This repository contains a project aiming to simplify Jenkins job 
creation (again).

It allows you to keep your jobs as a batch of template files that may 
be rendered down to Job DSL / Pipeline definitions any time you want.
And you'll probably want to add it as you main job to automatically
update your jobs on git push.

## Installation

```bash
gem install jenkins-importer
```

## How does it work

You may start with following structure:

```
jenkins-importer.yml
/build/.gitkeep
/src
  /organization
    /project
      /_context.yml
      /release-job.cadenza
      /release-job.yml
      /pipeline-job.cadenza
      /_partial.cadenza
```

And then call `jenkins-importer` inside that directory.

Importer will search for all `.cadenza`, `.twig` and `.liquid` files
that do not start with underscore (so `_partial.cadenza` will be 
omitted). After finding every file it will load context for each found 
file by following algorithm:

- Take relative template path
- Remove filename
- Split into chunks
- For each chunk concat path again, add `_context.yml`, then try to 
load it and merge with accumulator hash structure (so 
`release-job.cadenza` will try to load `src/_context.yml`, 
`src/organization/_context.yml` and 
`src/organization/project/_context.yml`)
- Change file extension to `.yml` and try to load it and merge with 
accumulator
- Try to find Jekyll-alike YAML front matter in file and merge it with
accumulator if present

After that if everything is good and nothing is disabled (see schema),
template is considered to be a job template and will be rendered to 
disk with same relative path into config `output` directory (with a 
simple exclusion for wrappers, described in following section), so you 
may have following structure:

```
/build
  /organization
    /project
      /release-job
      /_pipeline-job.wrap-source # see wrappers section
      /pipeline-job
```

After that importer takes list of rendered templates and throws it in
template specified in `aggregate.template` config option that renders
with same relative path in `output` directory specified by config.

Voila, now all you have to do is add Jenkins job that will import Job 
DSL from file and start pushing changes.

## Wrappers

Wrapper is simply a template that is applied to render result, so you 
may call `loadFileFromWorkspace` when needed (if you have ever tried 
to work with pipelines through job dsl plugin, you know that's a 
necessity). If wrapper is specified, render result is moved to
`_%original-name%.wrap-source`, and wrapper template will be
rendered under original name.

To specify a wrapper, make sure it is listed as `_template.wrapper` 
option in template context. In the example above, 
`context/organization/project/pipeline-job.yml` would have following
entry:

```yaml
_template:
  wrapper: pipeline.cadenza
```

There is only one wrapper available at the moment, `pipeline.cadenza`.
It exists for the sole purpose of dealing with pipeline scripts that 
have to be processed as regular strings by Job DSL constraints:

```twig
pipelineJob({% block job_id %}'{{ _job.name | default: _template.name }}'{% endblock %}) {
    {% block extension %}{% endblock %}
    {% block rotation %}
    logRotator {
        daysToKeep({{ _job.rotation.logs.days }})
        numToKeep({{ _job.rotation.logs.number }})
        artifactDaysToKeep({{ _job.rotation.artifacts.days }})
        artifactNumToKeep({{ _job.rotation.artifacts.number }})
    }
    {% endblock %}
    {% block definition %}
    definition {
        cps {
            script(readFileFromWorkspace('{{ _template.path }}'))
            sandbox(false)
        }
    }
    {% endblock %}
}
```

You may use any of your template files as wrapper, so to add some 
parameters to your pipeline job you would do the following:

```twig
{% extends 'pipeline.cadenza' %}
{# pipeline-override.cadenza #}
{% block extension %}
    parameters {
        booleanParam('FLAG', true)
        choiceParam('OPTION', ['option 1 (default)', 'option 2', 'option 3'])
    }
{% endblock %}
```

## Configuration schema

```yaml
# template location
sources: src
context: context
output: build
aggregate:
  disabled: false
  template: job-dsl.cadenza
  name: job-dsl
  context: {}
```

## Context schema

```yaml
_job:
  id: full job id, e.g. /organization/project/perform-minor-release
  name: human-readable job name, e.g. 'Perform minor release'
  description: human-readable job description
  rotation:
    artifacts:
      days: -1
      number: 30
    logs:
      days: -1
      number: 30
  disabled: |
    boolean, if set to true, template will be rendered, but job won't be 
    included into final set
_template:
  disabled: boolean, if set to true, template won't be rendered
  priority: integer marking the priority of template in render queue
  path: 
    # path to the rendered template, set automatically after template render
    relative: # relative from build location
    absolute:
```

## Roadmap

- Add support for jenkins environment variables to override context 
values 