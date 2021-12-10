---
layout: post
title:  "Python CI with GitHub Actions and Fedora"
permalink: /python-ci-with-github-actions-and-fedora/
categories: Projects
tags: 
  - CI
  - GitHub Actions
  - Python
  - Fedora
  - containers
---

Recently I started exploring continuous integration and even [created an example project.](https://github.com/regexowl/test-of-testing) With the basic knowledge of the topic, I started looking for an opportunity to implement a CI in "real life". Luckily I knew just the right person who was interested in setting up automated testing for one of his projects.

He introduced me to Content Resolver, a project under Fedora. And since I love open source I immediately took the opportunity. Soon I had [an issue assigned to me](https://github.com/minimization/content-resolver/issues/32) and was ready to dive head first into my first "real" CI project.

At the start we asked ourselves: "What is the most basic thing we can later build on?" We decided on setting up a workflow using Github Actions that will trigger on every pull request and simply run the full build.

From the start I was aware of some constraints and challenges:
- the build needs to run on Fedora
- there should be a place for adding more complex set of tests in the future


GitHub Actions only supports Ubuntu. But it can also run containers. So the simplest solution was to run all the jobs in the workflow in a Fedora container. The setup was quite quick:

```yaml
jobs:
  setup_and_test:
    runs-on: ubuntu-latest
    container:
      image: registry.fedoraproject.org/fedora:35
```


For now, the CI for the Content Resolver sets up a testing environment and runs the full build using a set of test configs. And to be able to add more tests in the future, I also added pytest to the workflow:

```yaml
- name: Run the build
  run: ./feedback_pipeline.py test_configs output
- name: Run tests
  run: |
    pytest
```


See [the merged pull request](https://github.com/minimization/content-resolver/pull/33) for a complete overview. It added the GitHub Actions workflow file and a python script `test_feedback_pipeline.py` which will serve as a main set of tests along the way.
