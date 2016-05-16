# cvmanager

For automation of some common tasks related to Content Views we created a tool called `cvmanager`. It consists of a Ruby script (`cvmanager`) and a YAML-formated configuration file (`cvmanager.yaml`). The various features are described in the following chapters.

## Cleanup of old Content Views

While working with Satellite 6, new Content View versions get created pretty often, especially when testing Puppet modules and having to make them available to the clients.
Old versions of the Content Views are kept by Satellite by default to be able to roll back to that specific version in the future. While this can be a very useful feature, keeping too many old versions makes the Satellite UI unclear and the background tasks slower as they tend to analyse more data.

To avoid this backlog of versions  we decided to automatically remove old Content View versions that are neither published nor part of a Composite Content View.

The `cvmanager` tool can execute this task when called as `cvmanager clean`. By default all Content View versions that are promoted to any Lifecycle Environment and the *5* newest non-promoted versions are kept. The number of non-promoted versions can be adjusted using the `--keep` commandline parameter or the `keep` entry in the configuration file.

`cvmanager clean` can be called from `cron` to fully automate this task.

## Automated Updates

You often present application-specific Composite Content Views to the Satellite clients. These Composite Content Views consist of Content Views for RHEL, common tools and Puppet modules and the specific application. Thus, when for example the RHEL Content View is updated with a new RHEL version or the Puppet Content View gets updated Puppet modules all Composite Content Views that use these Content Views need to be updated to point to the new Content View version.

`cvmanager` can ease this task by automatically updating Composite Content View contents to either a specific or latest version of a Content View. The call would be `cvmanager update`

First we need a configuration which version should be used:

    :cv:
      cv-RHEL7: 78.0
      cv-Puppet: latest
    :ccv:
      ccv-RHEL7-Capsule:
        cv-RHEL7: 77.0

The above configuration would ensure that all Composite Content Views that contain the "cv-RHEL7" Content View get it in version 78.0 but the Composite Content View for the Capsules will get the older 77.0 version (thus overriding the global setting of 78.0). It would also ensure that all Composite Content Views that contain "cv-Puppet" always get the latest version of it.

`cvmanager` will not make any asumtions about Content Views that are not listed in the configuration file, neither will it try to add or remove any Content Views to or from any Composite Content View, this is a manual task for the administator of the Satellite.

## Automated Promotes

After a new version of a Composite Content View is created it is only available in the Library Lifecycle Environment. To make it available to other environments a promotion of the version has to be done. We currently run the following Lifecycle order: Library > Development > Testing > Production. An automatic promotion to the first stage (Development) seems reasonable, while promotion to further stages should always happen manually by the administrator to not disturb the operation of the machines in these environments.

`cvmanager promote` will itterate over all Composite Content Views and check if their latest version is promoted to Development, if it isn't the promote is triggered.

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
    :cv:
      example_1: latest
      example_2: 28.0
    :ccv:
      application1:
        example_1: 29.0

* `user`: username of a Satellite 6 user to execute the actions with
* `pass`: password of the same user
* `uri`: URI of the Satellite 6, `https://localhost` will work when executed directly on the Satellite machine
* `timeout`: Timeout, in seconds, for any API calls made
* `org`: Organization ID (not name) for managing content in
* `livecycle`: target Lifecycle Environment ID (not name) for `promote`
* `keep`: how many non-published versions of a Content View shall be kept when doing a `clean`
