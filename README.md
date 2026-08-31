# Projet Kubernetes — Vaultwarden sur k3d

Deploiement d'un cluster Kubernetes en haute disponibilite hebergeant Vaultwarden (gestionnaire de mots de passe open source), avec autoscaling, observabilite, backup/restauration et GitOps.

## Stack technique

| Composant | Technologie |
|---|---|
| Cluster | k3d (k3s en conteneurs Docker) |
| Application | Vaultwarden + SQLite |
| Autoscaling | HPA + metrics-server |
| Observabilite | VictoriaMetrics + Grafana |
| Backup | Velero + Garage (S3) |
| GitOps | Flux v2 |
| Multicluster | Cluster API + provider Docker |

## Prerequis

- VMware Workstation (ou tout hyperviseur)
- VM Debian 12 - 8 Go RAM, 4 CPU, 50 Go disque
- Docker, kubectl, k3d, Helm

## Installation

### 1. Docker

    install -m 0755 -d /etc/apt/keyrings
    curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
    chmod a+r /etc/apt/keyrings/docker.gpg
    echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian bookworm stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
    apt update && apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

### 2. kubectl, k3d et Helm

    curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
    install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
    curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
    curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

### 3. Cluster k3d HA (3 control planes + 2 workers)

    k3d cluster create k8s-project --servers 3 --agents 2 --k3s-arg "--disable=traefik@server:*"
    kubectl get nodes

### 4. Vaultwarden

    kubectl apply -f vaultwarden/
    kubectl port-forward svc/vaultwarden 8080:80 -n vaultwarden --address 0.0.0.0 &
    # Accessible sur http://<IP_VM>:8080

### 5. Autoscaling HPA

    apt install -y hey
    kubectl port-forward svc/vaultwarden 8080:80 -n vaultwarden --address 0.0.0.0 &
    hey -n 50000 -c 100 http://localhost:8080
    kubectl get hpa -n vaultwarden

### 6. Observabilite — VictoriaMetrics + Grafana

    helm repo add vm https://victoriametrics.github.io/helm-charts/
    helm repo update
    helm install vm-stack vm/victoria-metrics-k8s-stack --namespace monitoring --create-namespace --set grafana.enabled=true --set vmsingle.enabled=true --set alertmanager.enabled=false --set vmalert.enabled=false --wait --timeout 5m
    kubectl get secret vm-stack-grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 --decode
    kubectl port-forward svc/vm-stack-grafana 3000:80 -n monitoring --address 0.0.0.0 &
    # Accessible sur http://<IP_VM>:3000 — login : admin

### 7. Backup — Velero + Garage

    git clone https://git.deuxfleurs.fr/Deuxfleurs/garage.git /tmp/garage
    helm install garage /tmp/garage/script/helm/garage/ --namespace backup --create-namespace --set garage.replicationFactor=1 --set garage.consistencyMode=degraded
    kubectl exec -n backup garage-0 -- ./garage status
    kubectl exec -n backup garage-0 -- ./garage layout assign -z dc1 -c 1G <ID_NODE_0>
    kubectl exec -n backup garage-0 -- ./garage layout assign -z dc1 -c 1G <ID_NODE_1>
    kubectl exec -n backup garage-0 -- ./garage layout assign -z dc1 -c 1G <ID_NODE_2>
    kubectl exec -n backup garage-0 -- ./garage layout apply --version 2
    kubectl exec -n backup garage-0 -- ./garage bucket create velero-backups
    kubectl exec -n backup garage-0 -- ./garage key create velero-key
    kubectl exec -n backup garage-0 -- ./garage bucket allow velero-backups --read --write --owner --key velero-key

    curl -Lo /tmp/velero.tar.gz https://github.com/vmware-tanzu/velero/releases/download/v1.15.0/velero-v1.15.0-linux-amd64.tar.gz
    tar -xf /tmp/velero.tar.gz -C /tmp/
    mv /tmp/velero-v1.15.0-linux-amd64/velero /usr/local/bin/

    velero install --provider aws --plugins velero/velero-plugin-for-aws:v1.11.0 --bucket velero-backups --secret-file /tmp/credentials-velero --use-volume-snapshots=false --backup-location-config region=garage,s3ForcePathStyle=true,s3Url=http://<GARAGE_IP>:3900

    # Backup
    velero backup create vaultwarden-backup --include-namespaces vaultwarden
    # Restauration
    kubectl delete namespace vaultwarden
    velero restore create --from-backup vaultwarden-backup

### 8. Bonus — GitOps avec Flux

    curl -s https://fluxcd.io/install.sh | bash
    export GITHUB_TOKEN=<TOKEN>
    flux bootstrap github --token-auth --owner=<USERNAME> --repository=k8s-project --branch=main --path=clusters/k8s-project --personal
    # Tout manifest dans clusters/k8s-project/ est applique automatiquement

### 9. Bonus — Multicluster avec Cluster API

    curl -L https://github.com/kubernetes-sigs/cluster-api/releases/download/v1.9.0/clusterctl-linux-amd64 -o /usr/local/bin/clusterctl
    chmod +x /usr/local/bin/clusterctl
    echo "fs.inotify.max_user_instances=1280" >> /etc/sysctl.conf
    echo "fs.inotify.max_user_watches=655360" >> /etc/sysctl.conf
    sysctl -p
    export CLUSTER_TOPOLOGY=true
    clusterctl init --infrastructure docker
    kubectl apply -f clusters/k8s-project/workload-cluster.yaml
    kubectl get clusters

Note : En environnement k3d, le provider CAPD ne peut pas acceder au socket Docker de la VM hote (contrainte Docker-in-Docker). Le cluster workload reste en Provisioning. En production, utiliser un provider cloud ou monter le socket hote avec --volume /var/run/docker.sock:/var/run/docker.sock@all.

## Structure du repo

    k8s-project/
    README.md
    vaultwarden/
        namespace.yaml
        pvc.yaml
        deployment.yaml
        service.yaml
        hpa.yaml
    clusters/
        k8s-project/
            flux-system/
            workload-cluster.yaml
            test-namespace.yaml
