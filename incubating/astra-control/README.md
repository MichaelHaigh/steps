# astra-control-toolkit

A Codefresh pipeline step to run [Astra Control Toolkit](https://github.com/NetApp/netapp-astra-toolkits/) commands to carry out common Kubernetes application data management operations. If you're not familiar with NetApp Astra Control, please view [this page](https://www.netapp.com/cloud-services/astra/) for more information.

## Shared Configuration Secret

A [shared configuration](https://g.codefresh.io/account-admin/account-conf/shared-config) secret YAML representing the Astra Control Toolkit `config.yaml` file **must** be created for this pipeline step to work correctly. This `config.yaml` file must contain Astra Control URL, account ID, and API token information in [the expected format](https://github.com/NetApp/netapp-astra-toolkits#authentication) for your Astra Control environment.

## Arguments

This pipeline step supports the following *pre-defined* arguments:

* `AC_CONFIG_SECRET`: the name of your [shared configuration secret](#shared-configuration-secret) representing your `config.yaml` file. The default value for this argument is `ac_config_yaml`, so this argument is only required if your shared secret name is different.
* `ACTOOLKIT_VERSION`: the [actoolkit](https://github.com/NetApp/netapp-astra-toolkits/tags) version to utilize (must be `>= 2.6.5`).
* `COMMANDS`: an array of any number of actoolkit (or general shell) commands to run. It is recommended to view the [actoolkit docs](https://github.com/NetApp/netapp-astra-toolkits/tree/main/docs#toolkit-functions), as there are a large number of commands and underlying argument permutations.

Additionally, this pipeline step supports any number of *custom* arguments, which are **all** mapped to environment variables. For example, you may want to set `APP_ID: b9451964-d651-486f-a80a-563d50bb0eaa` as an argument, which then allows for utilizing an environment variable `$APP_ID` in any of your `COMMANDS`.

## Examples

To create a backup of application `b9451964-d651-486f-a80a-563d50bb0eaa`:

```yaml
astra-control-toolkit:
  title: Astra Control Toolkit Create Backup
  type: astra-control-toolkit
  arguments:
    AC_CONFIG_SECRET: ac_config_yaml
    ACTOOLKIT_VERSION: 2.6.5
    APP_ID: b9451964-d651-486f-a80a-563d50bb0eaa
    BACKUP_NAME: cf-backup-`date "+%Y%m%d%H%M%S"`
    COMMANDS:
      - actoolkit create backup $APP_ID $BACKUP_NAME
```

The above example is functionally equivalent to this:

```yaml
astra-control-toolkit:
  title: Astra Control Toolkit Create Backup
  type: astra-control-toolkit
  arguments:
    AC_CONFIG_SECRET: ac_config_yaml
    ACTOOLKIT_VERSION: 2.6.5
    COMMANDS:
      - actoolkit create backup b9451964-d651-486f-a80a-563d50bb0eaa cf-backup-`date "+%Y%m%d%H%M%S"`
```

To create a snapshot:

```yaml
astra-control-toolkit:
  title: Astra Control Toolkit Create Snapshot
  type: astra-control-toolkit
  arguments:
    AC_CONFIG_SECRET: ac_config_yaml
    ACTOOLKIT_VERSION: 2.6.5
    APP_ID: b9451964-d651-486f-a80a-563d50bb0eaa
    SNAP_NAME: cf-snap-`date "+%Y%m%d%H%M%S"`
    COMMANDS:
      - actoolkit create snapshot $APP_ID $SNAP_NAME
```

To perform a live clone of a running application:

```yaml
astra-control-toolkit:
  title: Astra Control Toolkit Live Clone App
  type: astra-control-toolkit
  arguments:
    AC_CONFIG_SECRET: ac_config_yaml
    ACTOOLKIT_VERSION: 2.6.5
    APP_ID: b9451964-d651-486f-a80a-563d50bb0eaa
    CLONE_APP_NAME: cf-clone-`date "+%Y%m%d%H%M%S"`
    CLUSTER_ID: 928b9df2-9dd3-4cf6-ad80-ef5a2e546084
    COMMANDS:
      - >-
        actoolkit clone --sourceAppID $APP_ID --cloneAppName $CLONE_APP_NAME
        --clusterID $CLUSTER_ID
```

To create a snapshot of an application, and then clone from that snapshot:

```yaml
astra-control-toolkit:
  title: Astra Control Toolkit Snapshot and Clone App
  type: astra-control-toolkit
  arguments:
    AC_CONFIG_SECRET: ac_config_yaml
    ACTOOLKIT_VERSION: 2.6.5
    APP_ID: b9451964-d651-486f-a80a-563d50bb0eaa
    CLONE_APP_NAME: cf-clone-`date "+%Y%m%d%H%M%S"`
    CLUSTER_ID: 928b9df2-9dd3-4cf6-ad80-ef5a2e546084
    SNAP_NAME: cf-snap-`date "+%Y%m%d%H%M%S"`
    COMMANDS:
      - actoolkit create snapshot $APP_ID $SNAP_NAME
      - >-
        SNAP_ID=$(actoolkit -o json list snapshots -a $APP_ID | jq -r --arg
        SNAP_NAME "$SNAP_NAME" '.items[] | select(.name==$SNAP_NAME) | .id')
      - >-
        actoolkit clone --snapshotID $SNAP_ID --cloneAppName $CLONE_APP_NAME
        --clusterID $CLUSTER_ID
```

## Testing and Debugging

To test out this step locally, you can enter an interactive session with the following command (this assumes your `config.yaml` credential file is in your current working directory):

```sh
docker run -it -v ./config.yaml:/etc/astra-toolkits/config.yaml \
    netapp/astra-toolkits:2.6.5-minimal /bin/sh
```
