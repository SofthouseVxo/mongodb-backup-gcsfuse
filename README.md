# mongodb-backup

This image runs mongodump to backup data using cronjob to folder `/backup` mounted from a google store bucket using gcsfuse

Heavily influenced by [tutumcloud/mongodb-backup](github.com/tutumcloud/mongodb-backup) and [Requilence/mongodb-backup](github.com/Requilence/mongodb-backup)

## Usage:
### APIs and Service account
install [Google cloud SDK](https://cloud.google.com/sdk/) `gcloud` command line tool and run `gcloud init` to login and configure project settings.
Enable APIs

        gcloud service-management enable compute.googleapis.com

Create a service account with the [roles/storage.objectAdmin](https://cloud.google.com/storage/docs/access-control/iam-roles) role and generate a credentials key:

        NAME=mongodb-backup
        gcloud iam service-accounts create $NAME --display-name "$NAME"
        SERVICE_ACCOUNT=$(gcloud iam service-accounts list --filter=name:"$NAME" --format='value(email)')
        GCLOUD_PROJECT=$(gcloud config get-value project)
        gcloud projects add-iam-policy-binding $GCLOUD_PROJECT --member="serviceAccount:$SERVICE_ACCOUNT" --role="roles/storage.objectAdmin"
        gcloud iam service-accounts keys create google-application-credentials.json --iam-account=$SERVICE_ACCOUNT
        kubectl create secret generic google-application-credentials.json --from-file=google-application-credentials.json
### Storage Bucket
install the `gsutil` command line tool by executing `gcloud components install gsutil`
Create a [storage bucket](https://cloud.google.com/storage/docs/creating-buckets#storage-create-bucket-gsutil):

    gsutil mb gs://mybucket

### Usage

    apiVersion: extensions/v1beta1
    kind: Deployment
    metadata:
      name: mongodb-backup
    spec:
      replicas: 1
      template:
        metadata:
          labels:
            name: mongodb-backup
        spec:
          containers:
          - name: mongodb-backup
            image: jonaseck/mongodb-backup:3.4.5
            securityContext:
              privileged: true
              capabilities:
                add:
                  - SYS_ADMIN
            lifecycle:
              preStop:
                exec:
                  command: ["fusermount", "-u", "/backup"]
            imagePullPolicy: Always
            env:
            - name: MONGODB_HOST
              value: mongo
            - name: MONGODB_DB
              value: myDb
            - name: STORAGE_BACKUP_BUCKET_NAME
              value: mybucket
            - name: BACKUP_PREFIX
              value: test
            - name: GOOGLE_APPLICATION_CREDENTIALS
              value: "/credentials/google-application-credentials.json"
            volumeMounts:
            - name: google-application-credentials
              mountPath: "/credentials"
              readOnly: true
          volumes:
          - name: google-application-credentials
            secret:
              secretName: google-application-credentials

## Parameters

    MONGODB_HOST    the host/ip of your mongodb database
    MONGODB_PORT    the port number of your mongodb database
    MONGODB_USER    the username of your mongodb database. If MONGODB_USER is empty while MONGODB_PASS is not, the image will use admin as the default username
    MONGODB_PASS    the password of your mongodb database
    MONGODB_DB      the database name to dump. If not specified, it will dump all the databases
    MONGODB_RS      the replicaset name, adds --oplog and --oplogReplay to the backup and restore commands
    EXTRA_OPTS      the extra options to pass to mongodump command
    CRON_TIME       the interval of cron job to run mongodump. `0 0 * * *` by default, which is every day at 00:00
    MAX_BACKUPS     the number of backups to keep. When reaching the limit, the old backup will be discarded. No limit, by default
    INIT_BACKUP     if set, create a backup when the container launched

## Restore from a backup

See the list of backups, you can run:

    docker exec mongodb-backup ls /backup

To restore database from a certain backup, simply run:

    docker exec mongodb-backup /restore.sh /backup/2015.08.06.171901.gz

## Known issues
* Doesn't log to stdout due to tail -F not working on overlay file systems
* Silently fails if credentials are revoked

### Docker Usage
    docker run -d \
        --env MONGODB_HOST=mongodb.host \
        --env MONGODB_PORT=27017 \
        --env MONGODB_USER=admin \
        --env MONGODB_PASS=password \
        --env MONGO_RS=rs0 \
        --env GOOGLE_APPLICATION_CREDENTIALS=/credentials/google-application-credentials.json \
        --volume google-application-credentials.json:/credentials/google-application-credentials.json \
        jonaseck/mongodb-backup
