# cvmanager

For automation of some common tasks related to Content Views we created a tool called `cvmanager`. It consists of a Ruby script (`cvmanager`) and a YAML-formatted configuration file (`cvmanager.yaml`). The various features are described in the following chapters.

`cvmanager` is designed so that it can be run from `cron` or some other kind of scheduler easily.
Please remember to use only `labels` and not `names` when defining the Content Views or Composite Content Views in the configuration file.

## Cleanup of old Content Views

While working with Satellite 6, new Content View versions get created pretty often, especially when testing Puppet modules and having to make them available to the clients.
Old versions of the Content Views are kept by Satellite by default to be able to roll back to that specific version in the future. While this can be a very useful feature, keeping too many old versions makes the Satellite UI unclear and the background tasks slower as they tend to analyse more data.

To avoid this backlog of versions we decided to automatically remove old Content View versions that are neither published nor part of a Composite Content View.

The `cvmanager` tool can execute this task when called as `cvmanager clean`. By default all Content View versions that are promoted to any Lifecycle Environment and the *5* newest non-promoted versions are kept. The number of non-promoted versions can be adjusted using the `--keep` command-line parameter or the `keep` entry in the configuration file.


## Automated Publishes

Sometimes it can be beneficial to automatically update specific Content Views, for example the base OS Content View should be kept uptodate with the upstream OS repositories. To achieve that Katello/Satellite can schedule syncs of the repositories, but the synced content will not be available to the clients until the administrator publishes a new version of a Content View containing these repositories.

`cvmanager` can be configured to automatically publish new versions of Content Views if it detects changes to the contained repositories. Please note that the detection is only checking that the repository was synced after the last publish of the Content View, no verification is done whether the sync actually contained new packages. The option checkrepos will check the sync task and check if new packages were downloaded or not. If no new packages were downloaded, then the repository will not be marked for publish.

To use this feature, configure the `publish` array in the configuration file and call `cvmanager publish`.
The checkrepos option can be enabled using the `checkrepos` flag in the configuration file or by passing `--checkrepos` to `cvmanager publish`.


## Automated Updates

You often present application-specific Composite Content Views to the Satellite clients. These Composite Content Views consist of Content Views for RHEL, common tools, Puppet modules and the specific application. Thus, when for example the RHEL Content View is updated with a new RHEL version or the Puppet Content View gets updated Puppet modules all Composite Content Views that use these Content Views need to be updated to point to the new Content View version.

`cvmanager` can ease this task by automatically updating Composite Content View contents to either a specific or latest version of a Content View. The call would be `cvmanager update`.

First we need a configuration which version should be used:

    :cv:
      cv-RHEL7: 78.0
      cv-Puppet: latest
    :ccv:
      ccv-RHEL7-Capsule:
        cv-RHEL7: 77.0

The above configuration would ensure that all Composite Content Views that contain the "cv-RHEL7" Content View get it in version 78.0 but the Composite Content View for the Capsules will get the older 77.0 version (thus overriding the global setting of 78.0). It would also ensure that all Composite Content Views that contain "cv-Puppet" always get the latest version of it.

`cvmanager` will not make any assumptions about Content Views that are not listed in the configuration file, neither will it try to add or remove any Content Views to or from any Composite Content View, this is a manual task for the administrator of the Satellite.


## Automated Promotes

After a new version of a Composite Content View is created it is only available in the Library Lifecycle Environment. To make it available to other environments a promotion of the version has to be done. We currently run the following Lifecycle order: Library > Development > Testing > Production. An automatic promotion to the first stage (Development) seems reasonable, while promotion to further stages should always happen manually by the administrator to not disturb the operation of the machines in these environments.

`cvmanager promote` will iterate over all listed Composite Content Views and check if their latest version is promoted to Development, if it isn't the promote is triggered.


## Configuration

Example configuration for `cvmanager`:

    :settings:
      :user: admin
      :pass: changeme
      :uri: https://localhost
      :timeout: 300
      :org: 1
      :lifecycle: 2
      :keep: 5
      :wait: false
      :sequential: 1
    :cv:
      example_1: latest
      example_2: 28.0
    :ccv:
      application1:
        example_1: 29.0
    :publish:
      - example_1
    :promote:
      - application1

* `user`: username of a Satellite 6 user to execute the actions with
* `pass`: password of the same user
* `uri`: URI of the Satellite 6, `https://localhost` will work when executed directly on the Satellite machine
* `timeout`: Timeout, in seconds, for any API calls made
* `org`: Organization ID (not name) for managing content in
* `lifecycle`: target Lifecycle Environment ID (not name) for `promote`
* `keep`: how many non-published versions of a Content View shall be kept when doing a `clean`
* `wait`: should cvmanager wait for tasks to finish, or run them in the background
* `sequential`: how many tasks cvmanager can exec in parallel
* `checkrepos`: should cvmanager check that repo packages were actually downloaded before flagging for publish, or just publish if a sync executed
* `promote_cvs`: allow promotion of content-views


## Permissions

The following permissions are required to run `cvmanager`:

| Resource | Permissions |
|----------|-------------|
| Content Views | edit_content_views, view_content_views, publish_content_views, promote_or_remove_content_views, destroy_content_views|
| Lifecycle Environment | promote_or_remove_content_views_to_environments|
| Product and Repositories | view_products |
| Satellite tasks/task | view_foreman_tasks |

## Example Workflows

### Fully automated CCV for automated patching

#### Configuration

    :settings:
      :user: admin
      :pass: changeme
      :uri: https://localhost
      :org: 1
      :lifecycle: 2
    :ccv:
      ccv-RHEL7-automated:
        cv-RHEL7: latest
        cv-tools: latest
    :publish:
      - cv-RHEL7
    :promote:
      - ccv-RHEL7-automated

#### Execution

Using the above configuration, we could run the following script to achieve the fully automated CCV:

    #!/bin/sh

    set -e

    cvmanager --wait publish
    cvmanager --wait update
    cvmanager --wait promote

This would first publish a new version of `cv-RHEL7`, then update `ccv-RHEL7-automated` to contain the latest versions of `cv-RHEL7` and `cv-tools` and then promote the result to the Lifecycle Environment with id 2.

If needed, the `promote` step can be executed multiple times with `--to-lifecycle-environment` to push the CCV not only to "Development" but also to "Testing" or "Production":

    cvmanager --wait promote --to-lifecycle-environment 3
    cvmanager --wait promote --to-lifecycle-environment 4

### Fully automated CV for automated patching

When using CV instead of CCV

#### Configuration

    :settings:
      :user: admin
      :pass: changeme
      :uri: https://localhost
      :org: 1
      :lifecycle: 2
      :promote_cvs: true
    :cv:
      cv-RHEL7: latest
    :publish:
      - cv-RHEL7
    :promote:
      - cv-RHEL7

#### Execution

Same as the above example but without update since it's only related to CCVs

    #!/bin/sh

    set -e

    cvmanager --wait publish
    cvmanager --wait promote
