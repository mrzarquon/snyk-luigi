## Gromit

![gromit](https://raw.githubusercontent.com/mrzarquon/snyk-gromit/main/static/gromit.gif)

Experimental auto jobs / pipeline builder for snyk

This is a script (app/gromit.py) and a container that can run it in your pipeline.

gromit takes 2 inputs:
- folder of templates ending in .yaml to use for searches
- output name of the jobs file it should use

gromit builds the list of package files it should look for based on the *name* of templates in you provide:

package.json.yaml -> Gromit looks for package.json and uses package.json.yaml as the template to create a job for every folder that contains a file called 'package.json'

see these [examples](https://github.com/mrzarquon/snyk-job-templates)

It then builds a jobs file for you, which if you're using gitlab, you can then automatically generate child pipelines, letting you run snyk test/monitor in parallel for every project in your mono repo.

This uses GitLab CI's [dynamic child pipelines](https://docs.gitlab.com/ee/ci/parent_child_pipelines.html#dynamic-child-pipelines) feature

What does it look like?

When your pipeline kicks off it looks like this as Gromit is doing it's thing generating the jobs.yaml:
![first stage of pipeline](https://raw.githubusercontent.com/mrzarquon/snyk-gromit/main/static/first.png)

Then it spins up `trigger-snyk-tests` job runs, which loads the jobs.yaml and that creates the child pipeline containing a snyk test for each project folder it finds, based on those [templates](https://github.com/mrzarquon/snyk-job-templates) mentioned earlier:
![second stage showing the child pipeline running](https://raw.githubusercontent.com/mrzarquon/snyk-gromit/main/static/second.png)

And because we made sure to use the same `--remote-repo-url` and `--project-name=` using the same CI env variables in each job run, snyk was able to collate all of these individual project runs at snyk.io into a single project:
![Showing the snyk.io view](https://raw.githubusercontent.com/mrzarquon/snyk-gromit/main/static/third.png)

an example gitlab step that could use this is below:

```yaml
snyk-autogen:
  stage: .pre
  image: mrzarquon/gromit:latest
  script:
    - git clone --depth 1 https://github.com/mrzarquon/snyk-job-templates /tmp/templates
    - gromit /tmp/templates/gitlab jobs.yaml
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

scanning against a repo it generates a jobs.yaml, which the next step `trigger-snyk-tests` loads and that spins up the needed child pipelines
```yaml
snyk-node-frontend:
  image:
    entrypoint:
    - ''
    name: snyk/snyk:node-14
  script:
  - cd frontend
  - npm install
  - npm install snyk-to-html -g
  - snyk auth $SNYK_TOKEN
  - snyk monitor --remote-repo-url=${CI_PROJECT_URL} --project-name=${CI_PROJECT_NAMESPACE}/${CI_PROJECT_NAME}/frontend
  - snyk test --severity-threshold=high --remote-repo-url=${CI_PROJECT_URL} --project-name=${CI_PROJECT_NAMESPACE}/${CI_PROJECT_NAME}/frontend
  stage: build
snyk-python-backend/app:
  image:
    entrypoint:
    - ''
    name: snyk/snyk:python-3.8
  script:
  - cd backend/app
  - pip install poetry > /dev/null 2>&1
  - poetry install
  - snyk auth $SNYK_TOKEN
  - snyk monitor --remote-repo-url=${CI_PROJECT_URL} --project-name=${CI_PROJECT_NAMESPACE}/${CI_PROJECT_NAME}/backend/app
  - snyk test --severity-threshold=high --remote-repo-url=${CI_PROJECT_URL} --project-name=${CI_PROJECT_NAMESPACE}/${CI_PROJECT_NAME}/backend/app
  stage: build
```