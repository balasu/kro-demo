# kro-demo
#Setup GKE cluster ona project 
export CLUSTER_NAME=kro
export REGION=europe-west1
export PROJECT_ID=cloud-run-workshop-471907

gcloud services enable   container.googleapis.com    cloudresourcemanager.googleapis.com   serviceusage.googleapis.com   cloudkms.googleapis.com

gcloud container clusters create-auto $CLUSTER_NAME --region $REGION
gcloud container clusters get-credentials $CLUSTER_NAME --region $REGION

#KCC installation

gcloud storage cp gs://configconnector-operator/latest/release-bundle.tar.gz release-bundle.tar.gz

tar zxvf release-bundle.tar.gz

kubectl apply -f operator-system/configconnector-operator.yaml

gcloud iam service-accounts create kcc-operator

gcloud projects add-iam-policy-binding ${PROJECT_ID} --member="serviceAccount:kcc-operator@${PROJECT_ID}.iam.gserviceaccount.com"  --role="roles/owner"

gcloud iam service-accounts add-iam-policy-binding kcc-operator@${PROJECT_ID}.iam.gserviceaccount.com  --member="serviceAccount:${PROJECT_ID}.svc.id.goog[cnrm-system/cnrm-controller-manager]"  --role="roles/iam.workloadIdentityUser"

gcloud projects add-iam-policy-binding ${PROJECT_ID} --member="serviceAccount:kcc-operator@${PROJECT_ID}.iam.gserviceaccount.com"  --role="roles/storage.admin"

kubectl apply -f - <<EOF
apiVersion: core.cnrm.cloud.google.com/v1beta1
kind: ConfigConnector
metadata:
  name: configconnector.core.cnrm.cloud.google.com
spec:
  mode: cluster
  googleServiceAccount: "kcc-operator@${PROJECT_ID?}.iam.gserviceaccount.com"
  stateIntoSpec: Absent
EOF

kubectl wait -n cnrm-system --for=condition=Ready pod --all

#incase want to deploy resources on this namespace make sure to annotate
kubectl create namespace simple-web-app

kubectl annotate namespace simple-web-app  cnrm.cloud.google.com/project-id=${PROJECT_ID?}

#KRO installation

helm repo add kro [https://google.github.io/kro/](https://google.github.io/kro/)
helm repo update

export KRO_VERSION=$(curl -sL \
    https://api.github.com/repos/kubernetes-sigs/kro/releases/latest | \
    jq -r '.tag_name | ltrimstr("v")'
  )

echo $KRO_VERSION
helm install kro oci://registry.k8s.io/kro/charts/kro   --namespace kro   --create-namespace   --version=${KRO_VERSION}
kubectl get pods -n kro

kubectl get crd resourcegraphdefinitions.kro.run

#create simple webapp with ingress,pod,svc
 
kubectl apply -f rgd.yaml 
kubectl get rgd simple-rgd
kubectl apply -f instance.yaml 
kubectl get pods,svc,ingress
kubectl get ingress my-app-ingress -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
