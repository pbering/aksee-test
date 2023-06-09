# Azure Kubernetes Services Edge Essentials test

Testing AKS EE with Sitecore workload.

## Install AKS EE and deploy mixed Linux/Windows cluster on a single machine

Get and prepare downloads:
```powershell
Start-BitsTransfer "https://aka.ms/aks-edge/k8s-msi" -Destination "~\Downloads\k8s.msi"
Start-BitsTransfer "https://aka.ms/aks-edge/windows-node-zip" -Destination "~\Downloads\windows-node.zip"
Expand-Archive -Path "~\Downloads\windows-node.zip" -DestinationPath "~\Downloads"
```

Then start installation with:

```powershell
msiexec.exe /i (Get-Item "~\Downloads\k8s.msi").FullName /passiv ADDLOCAL=CoreFeature,WindowsNodeFeature
```

...or if you want the VM disks on another drive (see [docs](https://learn.microsoft.com/en-us/azure/aks/hybrid/aks-edge-howto-setup-machine#install-aks-edge-essentials) for all arguments):

```powershell
msiexec.exe /i (Get-Item "~\Downloads\k8s.msi").FullName /passiv ADDLOCAL=CoreFeature,WindowsNodeFeature VHDXDIR=D:\Data\AksEdge
```

Now verify you the installation was successful:

```powershell
Import-Module AksEdge
Get-Command -Module AKSEdge | Format-Table Name, Version
```

> IMPORTANT! Before starting the cluster deployment, please notice that the following command will **overwrite** your current [kubeconfig](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/) at $HOME/.kube/config, so make sure to backup if needed!

Now you can deploy a new cluster (takes about ~6 minutes):

```powershell
New-AksEdgeDeployment -JsonConfigFilePath ".\single-mixed.json"
```

You now have a running Kuberneters single machine cluster with a Linux node and Windows node! You can verify with `kubectl get node`.

## Post deployment steps

1. Taint the Windows node: `kubectl taint node "$(Get-AksEdgeNodeName -NodeType Windows)" os=windows:NoSchedule`
1. Install nginx ingress:
    - Install: `kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.0/deploy/static/provider/baremetal/deploy.yaml`
    - Configure nginx with: `kubectl edit svc ingress-nginx-controller -n ingress-nginx`, change `spec.type` from `NodePort` to `LoadBalancer`.
    - Verify that nginx got an external IP address: `kubectl get services -n ingress-nginx ingress-nginx-controller`
1. Install local path provisioner for Linux container persistence:
    - Run: `kubectl apply -f https://raw.githubusercontent.com/Azure/AKS-Edge/main/samples/storage/local-path-provisioner/local-path-storage.yaml`
    - For details, see [docs](https://learn.microsoft.com/en-us/azure/aks/hybrid/aks-edge-howto-use-storage-local-path)...

## Deploy Sitecore XM

1. Create a new project directory: `mkdir .\sitecore-xm;cd .\sitecore-xm`
1. Download workload: `curl.exe https://raw.githubusercontent.com/pbering/aksee-test/main/test/xm/kustomization.yaml -o .\kustomization.yaml`
1. Generate certificates (download mkcert <https://github.com/FiloSottile/mkcert/releases>):

    ```powershell
    mkcert -install
    mkcert -cert-file ".\cm-tls.crt" -key-file ".\cm-tls.key" "cm.aksee.local"
    mkcert -cert-file ".\id-tls.crt" -key-file ".\id-tls.key" "id.aksee.local"
    ```

1. Prepare HOST file, run `kubectl get services -n ingress-nginx ingress-nginx-controller` and use the external IP to add two new entries:
      - `192.168.1.4  id.aksee.local`
      - `192.168.1.4  cm.aksee.local`

1. Prepare Sitecore license, run:

   ``` powershell
   Import-Module SitecoreDockerTools
   ConvertTo-CompressedBase64String -Path "C:\License\license.xml" | Out-File -Encoding ascii -NoNewline -FilePath .\sitecore-license.txt
   ```

1. Then deploy the workload:

    ```powershell
    kubectl create namespace xm
    kubectl apply -n xm -k .
    ```

1. You can observe the progress with `kubectl -n xm get pods` or <https://k9scli.io/> or just wait with `kubectl wait --for=condition=complete job mssql-init solr-init -n xm --timeout 10m`
1. Done! Now you can visit <https://cm.aksee.local>.
