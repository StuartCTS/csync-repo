Install operator:

 ```
 gsutil cp gs://config-management-release/released/latest/config-management-operator.yaml ~/tmp/config-management-operator.yaml
 
 kubectl apply -f ~/tmp/config-management-operator.yaml
 ```


 switch contexts (after getting creds via gcloud):

 ```
 kubectl config use-context gke_stuart-dev-example-01_europe-west1-b_belgium
 kubectl config use-context gke_stuart-dev-example-01_us-central1-c_iowa
 ``

 kubectl get clusterrolebinding auditors