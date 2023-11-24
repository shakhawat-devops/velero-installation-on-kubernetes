Hi! As companies move to microservices container orchestration is becoming the most sought-after task nowadays. As part of this more companies are adopting Kubernetes, known as also K8s as their container orchestration tool. Since Kubernetes has become the only tool to orchestrate containers it is important to implement a backup and restore methodology for backing up the Kubernetes cluster and resources hosted on it so that organizations can withstand any disaster. 

In this article, I will try to implement such a backup & restore methodology in order to successfully backup and restore our Kubernetes cluster. Though we can take the backup of the ETCD database, implementing a solution that will periodically take the backup of the whole cluster and save it to an Object store like S3 could make our Kubernetes cluster more robust and fault-tolerant. In my demo I will use Velero: an open-source tool for backing and restoring Kubernetes resources and an Object storage Minio: an open-source object storage with the functionality of S3, to save the backed up and restored objects there. 

Let's begin!

In my demo, I will first implement the Object storage: Minio as a Docker container which will run in my environment locally. Let's jump into it. 

### Installing Minio as Docker Container

First I will run a docker container with Minio image in my environment:
   
   ``` 
   docker run -d --name minio \
   -p 9000:9000 \
   -p 9001:9001 \
   -e "MINIO_ROOT_USER=<your-user-name>" \
   -e "MINIO_ROOT_PASSWORD=<your-password>" \
   quay.io/minio/minio server /data --console-address ":9001"
  ```
Let's explain the commands briefly: 

Here first I am allocating port numbers 9000 and 9001 on which Minio listens. Then I am passing two environment variables where I have mentioned the initial login user and password. Then I pulled the image of Minio from quay.io and started the server using the "server /data" command which will start the Minio server to access. When the Minio docker container starts running login into it and create a bucket named "k8sbackup"


### Installing Velero

Now let's start installing the component that will enable us to take a backup of our Kubernetes cluster and restore it when needed. The name of the tool is Velero. It is an open-source tool that was formerly known as heptio Ark. We can use Velero with any cloud provider or on-premise. It gives us three benefits. They are: 

  1. Take backups of the cluster and restore in case of loss.
  2. Migrate cluster resources to other clusters.
  3. Replicate production clusters to development and testing clusters.

To install it in the Kubernetes cluster first we need to download the binary and then place them into the bin directory. This will install the Velero command line utility and after that, we will install it using the Velero install command. 

### Installing the Velero cmd
1. Download the latest release from github: https://github.com/vmware-tanzu/velero/releases/tag/v1.6.3
2. Extract the tarball: tar -xvf <RELEASE-TARBALL-NAME>.tar.gz
3. Move the extracted velero binary to somewhere in your $PATH (/usr/local/bin for most users) 

After that you will be able to run the Velero command using the Velero CLI!

But wait! We're not done yet. We will have to deploy the velero as a POD into our Kubernetes cluster. 

Velero uses a storage provider plugin to save the backup to the object storage. There are different storage providers velero can support and for a particular object storage we need to use the appropriate storage plugin. For details please see the link: https://velero.io/docs/v1.6/supported-providers/

For our use case, we are using S3-compatible object storage provider Minio and for that, we need to use "velero-plugin-for-aws:v1.2.0". To successfully use the storage provider velero needs the credentials to connect to the bucket created in Minio and for that, we need to pass them using a config file. 

Create a config file named: ``credentials-velero`` and add the below texts into it: 
```
  [default]
  aws_access_key_id = <your-user-name>
  aws_secret_access_key = <your-passsword>
```
After that use the below command to run the Velero pod in your Kubernetes cluster: 
```
velero install \
    --provider aws \
    --plugins velero/velero-plugin-for-aws:v1.2.0 \
    --bucket <bucket_name> \
    --secret-file <fullpath velero config file created earlier> \
    --use-volume-snapshots=false \
    --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://<your_MinIO_server_IP>:9000
```

One thing to note here is that we are using the volume-snapshot flag false in our demo. That means we will not use any volume snapshot feature by Velero but in the production, this flag must be set to true. 

Now Check your velero deployment is successfull or not. Velero creates a separate namespace to isolate its resources. Check all the pods are up or not by kubectl get pods -n velero. 

With this command velero will CLI will create the necessary objects including some custom resources defition (CRDs) into your kubernetes cluster. To view the CRDs run: kubectl get crd -n velero. 

That's it! We have successfully deployed Velero into our kubernetes cluster. We are now ready to take backup of any objects of our cluster and save it to the Minio Object storage. 



### Take backup of a namespace including all the objects
1. To test the backup and restore functionality of Velero let's create a namespace named "app" and deploy an nginx pod and loadbalancer service to expose it.
   ```
   kubectl run nginx --image=nginx -n app
   kubectl expose pod nginx --name=nginx-web --port=80 --type=LoadBalancer
   ```
   
2. Now run: 
   ```
   velero create backup nginx-backup --include-namespaces app
   ```

This command will take a backup of the app namespace and will log a message in the command line. To verify that use the Minio dashboard and you will find a backup file in the bucket you have mentioned. Now Delete the namespace and then restore the backup from the Minio server. 

1. Deleting the namespace: ``kubectl delete ns app``
2. Restoring the backup: ``velero restore create <restore_name> --from-backup <backup-name>``

Note that velero creates a restore object for this and this object will also be saved in the Object storage. If we want to delete a particular backup and restore object we can run: velero delete backup <backup-name> or velero delete restore <restore-name> 

We can also schedule the backup using "velero schedule create" command which will use a cron expression to take the backup of our choise in a timely manner. 



This is most basic setup but for any production use we have to plan it accordingly. We have to use a persistent storage for our Minio Server so that we can persist the backup and restore data. Also for all the kubernetes deployment it is necessary to use Persistent Volumes (PV) in k8s so that our deployment does not dependet on node local storage. 
