# Status monitoring


You can monitor the status of your database cluster by running the following command:

```sh
kubectl get databasecluster <cluster-name> -o jsonpath='{.status}'
```

The `status` field will show one of these states:

- **creating**: Initial creation in progress
- **initializing**: Database is being initialized
- **ready**: Database is running and ready
- **error**: An error has occurred
- **paused**: Database is paused — all pods are stopped, storage is preserved. See [Pause and resume databases](../use/pause_resume.md).
- **upgrading**: Database is being upgraded
- **restoring**: Database is being restored from backup
