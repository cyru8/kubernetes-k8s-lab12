# kubernetes-k8s-lab12

Velero gives you tools to back up and restore your Kubernetes cluster resources and persistent volumes. Velero lets you:

    Take backups of your cluster and restore in case of loss.
    Copy cluster resources across cloud providers. NOTE: Cloud volume migrations are not yet supported.
    Replicate your production environment for development and testing environments.

Velero consists of:

    A server that runs on your cluster
    A command-line client that runs locally

For more information, the documentation provides a getting started guide, plus information about building from source, architecture, extending Velero, and more.
######

Set Up our Environment

Let's pull down the Heptio Velero GitHub repo to help us get started: git clone https://github.com/heptio/velero

Let's download and install the Velero client: curl -LO https://github.com/heptio/velero/releases/download/v1.1.0/velero-v1.1.0-linux-amd64.tar.gz

tar -C /usr/local/bin -xzvf velero-v1.1.0-linux-amd64.tar.gz

Add velero directory to PATH export PATH=$PATH:/usr/local/bin/velero-v1.1.0-linux-amd64/

Create a Velero-specific credentials file (credentials-velero) in your local directory:

echo "[default]
aws_access_key_id = minio
aws_secret_access_key = minio123" > credentials-velero

Start the server and the local storage service:

kubectl apply -f velero/examples/minio/00-minio-deployment.yaml

velero install \
    --provider aws \
    --bucket velero \
    --secret-file ./credentials-velero \
    --use-volume-snapshots=false \
    --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://minio.velero.svc:9000

This example assumes that it is running within a local cluster without a volume provider capable of snapshots, so no VolumeSnapshotLocation is created (--use-volume-snapshots=false).

Deploy an example nginx application: kubectl apply -f velero/examples/nginx-app/base.yaml

Check to see that both Velero and nginx deployments have been successfully created:

kubectl get deployments -l component=velero --namespace=velero

kubectl get deployments --namespace=nginx-example

######

Back Up Our Resources

Let's backup any resource with labels "app=nginx": 

velero backup create nginx-backup --selector app=nginx

Verify if the backup has completed velero backup describe nginx-backup

Now, let's simulate a disaster: 

kubectl delete namespace nginx-example

Check that the nginx service and deployment are gone:

kubectl get deployments --namespace=nginx-example

kubectl get services --namespace=nginx-example

kubectl get namespace/nginx-example

You should get no results.

NOTE: You might need to wait for a few minutes for the namespace to be fully cleaned up.

######
Restore Our Resources

Run: velero restore create --from-backup nginx-backup

Run: velero restore get

NOTE: The restore can take a few moments to finish. During this time, the STATUS column reads InProgress.

After a successful restore, the STATUS column is Completed, and WARNINGS and ERRORS are 0. All objects in the nginx-example namespace should be just as they were before you deleted them.

If there are errors or warnings, you can look at them in detail:

velero restore describe <RESTORE_NAME>

You can verify that the nginx resources are available again:

kubectl get services --namespace=nginx-example

kubectl get namespace/nginx-example

######

