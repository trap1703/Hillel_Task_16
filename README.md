gcloud config set compute/zone us-central1

cd kubernetes-engine-samples/wordpress-persistent-disks

WORKING_DIR=$(pwd)

CLUSTER_NAME=persistent-disk-tutorial

gcloud container clusters create $CLUSTER_NAME \
    --num-nodes=1 --enable-autoupgrade --no-enable-basic-auth \
    --no-issue-client-certificate --enable-ip-alias --metadata \
    disable-legacy-endpoints=true
kubectl create -f lbg.yaml --namespace=monitoring

kubectl apply -f $WORKING_DIR/wordpress-volumeclaim.yaml

INSTANCE_NAME=mysql-wordpress-instance1
gcloud sql instances create $INSTANCE_NAME

export INSTANCE_CONNECTION_NAME=$(gcloud sql instances describe $INSTANCE_NAME \
    --format='value(connectionName)')

CLOUD_SQL_PASSWORD=$(openssl rand -base64 18)
gcloud sql users create wordpress --host=% --instance $INSTANCE_NAME \
    --password $CLOUD_SQL_PASSWORD

SA_NAME=cloudsql-proxy
gcloud iam service-accounts create $SA_NAME --display-name $SA_NAME

SA_EMAIL=$(gcloud iam service-accounts list \
    --filter=displayName:$SA_NAME \
    --format='value(email)')

gcloud projects add-iam-policy-binding $DEVSHELL_PROJECT_ID \
    --role roles/cloudsql.client \
    --member serviceAccount:$SA_EMAIL
	
gcloud iam service-accounts keys create $WORKING_DIR/key.json \
    --iam-account $SA_EMAIL

kubectl create secret generic cloudsql-db-credentials \
    --from-literal username=wordpress \
    --from-literal password=$CLOUD_SQL_PASSWORD

kubectl create secret generic cloudsql-instance-credentials \
    --from-file $WORKING_DIR/key.json

cat $WORKING_DIR/wordpress_cloudsql.yaml.template | envsubst > \
    $WORKING_DIR/wordpress_cloudsql.yaml

kubectl create -f $WORKING_DIR/wordpress_cloudsql.yaml

kubectl get pod -l app=wordpress --watch

kubectl create -f $WORKING_DIR/wordpress-service.yaml 

kubectl get svc -l app=wordpress --watch

=============End part1 ========

kubectl create namespace monitoring

kubectl create -f clusterRole.yaml

kubectl create -f config-map.yaml

kubectl create  -f prometheus-deployment.yaml 

kubectl get deployments --namespace=monitoring

kubectl create -f lb.yaml --namespace=monitoring

=============End part2 ========

kubectl create -f grafana-datasource-config.yaml

kubectl create -f deployment.yaml

kubectl create -f service.yaml

kubectl create -f lbg.yaml --namespace=monitoring

User: admin
Pass: admin
