---
layout: post
title:  "GitHub Actions CI with an output to a comment for pull request"
permalink: /github-actions-ci-with-an-output-to-a-comment-for-pull-request/
categories: Projects
tags: 
  - CI
  - GitHub Actions
  - Python
  - Fedora
  - containers
  - GitHub API
---
The Content Resolver project under Fedora consists of two repositories, one of them contains all the code and the second one is where the input configuration files live. People can contribute to the project and use it by uploading new configuration files. Sometimes the configs can be incomplete or include errors, this is where an opportunity to set up a CI comes in.

I have implemented a CI with GitHub Actions which checks the config files for possible problems using a function already implemented in the code repository. 

The whole idea is that in the [code repository](https://github.com/minimization/content-resolver), there is a short script `test_config_files.py` added, that calls only one function from the project. Function `get_configs()` which is already implemented in the `feedback_pipeline.py` and which checks if the configuration files are valid and complete. This function is run with a set of mock settings defined in the short script. The results of the function are then parsed within the GitHub Actions workflow and output as a PR comment. 

The maintainer of the project and the contributor are within few minutes able to see, if there’s anything that would need to be fixed.

From the start I needed to work out a few technicalities:
- there are **two repositories** (input one with the configs and then one with the main code) and there shouldn't be any duplicities in the code
- the results of the workflow run should be put **into a PR comment**


The setup uses a Fedora container run under the Ubuntu runner.

```yaml
jobs:
  setup_and_test:
    runs-on: ubuntu-latest
    container: 
      image: registry.fedoraproject.org/fedora:35
```


There are two repositories which I need for this workflow to run. With the [code repository](https://github.com/minimization/content-resolver) containing the [input repository](https://github.com/minimization/content-resolver-input) with configuration files. I’ve used checkout with a specified path to nest them as needed. 

```yaml
- name: Checkout code repo
  uses: actions/checkout@v2
  with:
    repository: minimization/content-resolver
    path: code
    
- name: Checkout
  uses: actions/checkout@v2
  with:
    path: code/input
```


Up to this point everything seemed quite straightforward, but the real fun was coming. The only thing left was to output the results of the `get_configs()` into a comment under the pull request. But after a test run I’ve noticed the workflow never output more than the first line of the results. 

After some experiments I found out, that working with multiline strings in GitHub Actions is a bit problematic. Whitespace characters are parsed in a way that it wasn’t possible to output a block of text. When searching for a solution, I've stumbled upon [this blogpost](https://trstringer.com/github-actions-multiline-strings/) introducing several options to bypass this problem.

The solution I chose in the end was sanitisation of the string or string substitution since it's close to regex and I couldn’t possibly pass on that.

The contents of the variable `RESULTS` (multiline string) is url encoded and then returned which makes it one really long string for the workflow, but in the final http encoding it looks just like needed.

```yaml
- name: Run get_configs and set output
  run: |
    cd code
    RESULTS=$(python3 test_config_files.py 2>&1)
    RESULTS="${RESULTS//'%'/'%25'}"
    RESULTS="${RESULTS//$'\n'/'%0A'}"
    RESULTS="${RESULTS//$'\r'/'%0D'}"
    echo "::set-output name=PR_COMMENT::$RESULTS"
  id: get_configs_output
```


Finally I needed to print the output into a PR comment. For this there's an easy solution using GitHub API. The only trickier part was showing the output as a block of code in the comment itself. Since javascript uses backticks to define a multiline comment and the block of code in markdown syntax also uses backticks, I needed to nest them with some strategic escaping.

```yaml
- name: Comment on the PR
  uses: actions/github-script@v5
  with: 
    github-token: ${{ secrets.GITHUB_TOKEN }}
    script: |
      github.rest.issues.createComment({
        issue_number: context.issue.number,
        owner: context.repo.owner,
        repo: context.repo.repo,
        body: `\`\`\`\n${{ steps.get_configs_output.outputs.PR_COMMENT }}\n\`\`\``   
      })
```


You can see the resulting PR comment of the workflow [in this demonstration](https://github.com/regexowl/content-resolver-input/pull/50).
