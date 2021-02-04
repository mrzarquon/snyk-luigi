Experimental auto jobs / pipeline builder for snyk

This is a script (app/luigi.py) and a container that can run it in your pipeline.

luigi takes 2 inputs:
folder of templates ending in .yaml to use for searches
output name of the jobs file it should use

luigi builds the list of package files it should look for based on the *name* of templates in you provide:

package.json.yaml -> Luigi looks for package.json and uses package.json.yaml as the template to create a job for every folder that contains a file called 'package.json'

see these [examples](https://github.com/mrzarquon/snyk-job-template)

It then builds a jobs file for you, which if you're using gitlab, you can then automatically generate child pipelines, letting you run snyk test/monitor in parallel for every project in your mono repo.

This uses GitLab CI's [dynamic child pipelines](https://docs.gitlab.com/ee/ci/parent_child_pipelines.html#dynamic-child-pipelines) feature

an example gitlab step that could use this is below:

```yaml
snyk-autogen:
  stage: .pre
  image: mrzarquon/luigi:latest
  script:
    - git clone --depth 1 https://github.com/mrzarquon/snyk-job-templates /tmp/templates
    - luigi /tmp/templates/gitlab jobs.yaml
  artifacts:
    paths:
      - jobs.yaml

trigger-snyk-tests:
  stage: build
  needs:
    - snyk-autogen
  trigger:
    include:
      - artifact: jobs.yaml
        job: snyk-autogen
    strategy: depend
```