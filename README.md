# ArgoCD Installation Guide for Windows with Rancher Desktop

## Introduction to ArgoCD
ArgoCD is a declarative, GitOps continuous delivery tool for Kubernetes. It provides a simple and intuitive web UI to manage and deploy applications to a Kubernetes cluster. With ArgoCD, you can automate your application deployments, rollbacks, and updates, ensuring consistency and reliability in your Kubernetes environment.

## Prerequisites
Before installing ArgoCD on your Windows machine with Rancher Desktop, make sure you have the following prerequisites:
- Rancher Desktop installed and running on your Windows machine. For that to work, you may need to enable Hyper-v in system features and apps
- A Kubernetes cluster provisioned and running within Rancher Desktop.

## Steps to Install ArgoCD
Follow these steps to install ArgoCD on your Windows machine with Rancher Desktop:

1. Open a terminal or command prompt.

2. Create a namespace called argocd
    ```shell
    kubectl create namespace argocd
    ```

3. Install ArgoCD directly from official project repository:
    ```shell
    kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
    ```

4. Once installation is done, it will create some workloads and ArgoCD server should run in the `argocd` namespace.
   Verify that the ArgoCD components are running:
    ```shell
    kubectl get pods -n argocd
    ```

5. Patch the service to change to LoadBalancer
    ```shell
    kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
    ```
    
6. Config ArgoCD CLI with username `admin` and password `admin`
    
    ```shell
    kubectl -n argocd patch secret argocd-secret \
    -p '{"stringData": {"admin.password": "$2a$10$mivhwttXM0U5eBrZGtAG8.VSRL1l9cZNAmaSaqotIzXRBRwID1NT.",
        "admin.passwordMtime": "'$(date +%FT%T)'"
    }}'
    ```
7. Do the port forward from ArgoCD application to your localhost 
    ```shell
    kubectl port-forward svc/argocd-server -n argocd 10443:443 2>&1 > /dev/null &
    ```
    You can now access the ArgoCD web UI by opening a web browser and navigating to `https://localhost:10443`.

9. Log in to the ArgoCD web UI using the default username and password:
    - Username: `admin`
    - Password: `admin`
    It is recommended to change the default password after logging in. In the page 
    `https://localhost:10443/user-info`

10. Install argocd cli utility to connect you any ArgoCD connection from CLI. Follow instructions in [URL](https://argo-cd.readthedocs.io/en/stable/cli_installation/#windows) in `powershell`. Don't forget to change the path of your your username in `<yourusername>` where `argocd.exe` is copied.

    ```shell
    cd ~
    $version = (Invoke-RestMethod https://api.github.com/repos/argoproj/argo-cd/releases/latest).tag_name
    $url = "https://github.com/argoproj/argo-cd/releases/download/" + $version + "/argocd-windows-amd64.exe"
    $output = "argocd.exe"
    Invoke-WebRequest -Uri $url -OutFile $output
    [Environment]::SetEnvironmentVariable("Path", "$env:Path;C:\Users\<yourusername>", "User")
    ```
    
11. Use a sample application `guestbook_app`. Create a name space in your cluster

    ```shell
     kubectl create ns guestbook
    ```
    After creating the name space create the application as project in ArgoCD using command line.
    you can refer to `argoproj_application.yaml`. Remember to replace the github URL in the file.
    ```shell
    kubectl apply -f argoproj_application.yaml
    ```

    or run this command in your cli to create on the fly.
    

    ```shell
    # Get the Github_URL something like this: https://github.com/sudhakarm/argocd-poc.git
    HTTPS_GITHUB_URL=$(git remote show origin | sed -n 's/.*URL: \(https:\/\/github.com\/[^ ]*\).*/\1/p' | head -n 1)
    
    # Apply application
    cat <<EOF | kubectl apply -f -
    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
    name: guestbook
    namespace: argocd
    spec:
    destination:
        namespace: guestbook
        server: https://kubernetes.default.svc
    project: default
    source:
        repoURL: $HTTPS_GITHUB_URL
        path: guestbook_app
        targetRevision: master
    EOF
    ```

12. After deploying application, check the ArgoCD UI in browser. You should be able to see new application in default project `https://localhost:10443/applications`

13. Now make a change to the guestbook_app to see changes reflecting in the argoCD