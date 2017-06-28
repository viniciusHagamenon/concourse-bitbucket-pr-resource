# concourse-bitbucket-pr-resource

Tracks changes for all *git* branches for pull requests.

**This resource is meant to be used with [`version:
every`](https://concourse.ci/get-step.html#get-version).**

Inspirited by [git-branch-heads-resourse](https://github.com/vito/git-branch-heads-resource)

## Installation

Add the following `resource_types` entry to your pipeline:

```yaml
---
resource_types:
- name: git-bitbucket-pr
  type: docker-image
  source: {repository: viniciushagamenon/concourse-bitbucket-pr-resource}
```

## Source Configuration

* `base_url`: *Required*. base URL of the bitbucket server, without a trailing slash. 
For example: `http://bitbucket.local`
* `username`: *Required*. username of the user which have access to repository.
* `password`: *Required*. password of that user
* `project`: *Required*. project for tracking
* `repository`: *Required*. repository for tracking
* `limit`: *Optional*. limit of tracked pull requests `default: 100`.
* `git`: *Required*. configuration is based on the [Git
resource](https://github.com/concourse/git-resource). The `branch` configuration
from the original resource is ignored.
* `bitbucket_type`: *Optional*. `cloud` for BitBucket Cloud or `server` for a self-hosted BitBucket Server. `default: server`
* `dir`: *Optional*. set to name of the resource if resource name is different than repository name

### Example

``` yaml
resources:
- name: my-repo-with-pr
  type: git-bitbucket-pr
  source:
    bitbucket_type: cloud
    url: https://bitbucket.org
    username: some-username
    password: some-password
    project: viniciusHagamenon
    repository: concourse-bitbucket-pr-resource
    git:
      uri: https://github.com/viniciusHagamenon/concourse-bitbucket-pr-resource
      private_key: ((git-repo-key))

jobs:
  - name: unit
    plan:
      - get: my-repo-with-pr
        trigger: true
        version: every

      # update pr with build status - INPROGRESS
      - put: my-repo-with-pr
        params:
          name: My CI                 # name of the build displayed at the pr on bitbucket
          description: CI unit tests  # description of the build displayed at the pr on bitbucket
          state: INPROGRESS           # bitbucket build state - [INPROGRESS, SUCCESSFUL, FAILED]

      - task: unit-test
        file: my-repo-with-pr/ci/unit.yml # your task file

        on_success: # update pr with build status - SUCCESSFUL
          put: my-repo-with-pr
          params:
            name: My CI
            description: CI unit tests
            state: SUCCESSFUL

        on_failure: # update pr with build status - FAILED
          put: my-repo-with-pr
          params:
            params:
            name: My CI
            description: CI unit tests
            state: FAILED
```

## Behavior

### `check`: Check for changes to all pull requests.

The current open pull requests fetched from Bitbucket server for given 
project and repository. Update time are compared to the last fetched pull request.

If any pull request are new or updated or removed, a new version is emitted.

### `in`: Fetch the commit that changed the pull request.

This resource delegates entirely to the `in` of the original Git resource, by
specifying `source.branch` as the branch that changed, and `version.ref` as the
commit on the branch.

All `params` and `source` configuration of the original resource will be
respected.

### `out`: Update build task status.

This updates the build status of the task.