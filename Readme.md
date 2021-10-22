From GIT to Kubernetes in 10 minutes with ArgoCD.

# Source: https://santanderglobaltech.com/en/from-git-to-kubernetes-in-10-minutes-with-argocd/

Step 0: Requirement (which i apply)
a) docker desktop with k8s enabled
b) github 
c) argocd dli

# Step 1: Install ArgoCD – 1 min
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Step 2: Configure ArgoCD – 2 min
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
kubectl -n argocd patch secret argocd-secret \
    -p '{"stringData": {"admin.password": "$2a$10$mivhwttXM0U5eBrZGtAG8.VSRL1l9cZNAmaSaqotIzXRBRwID1NT.",
        "admin.passwordMtime": "'$(date +%FT%T)'"
    }}'
argocd login localhost:10443 --username admin --password admin --insecure
kubectl port-forward svc/argocd-server -n argocd 10443:443 2>&1 > /dev/null &

UI: https://localhost:10443

# Step 3: Create a deployment GIT repository – 2 min
gh repo create --public git2k8s10min -y && cd git2k8s10min
echo "From GIT to Kubernetes in 10 minutes with ArgoCD." > Readme.md
git add Readme.md
git commit -m "Added Readme.md" && git push --set-upstream origin master
git branch live
git push --set-upstream origin live
git checkout master
curl -kSs https://raw.githubusercontent.com/kubernetes/examples/master/guestbook/all-in-one/guestbook-all-in-one.yaml -o guestbook_app.yaml
git add guestbook_app.yaml
git commit -m "Added guestbook_app.yaml"
git push --set-upstream origin master

# Step 4: Deploy using ArgoCD – 3 min
HTTPS_REPO_URL=$(git remote show origin |  sed -nr 's/.+Fetch URL: git@(.+):(.+).git/https:\/\/\1\/\2.git/p')
kubectl create namespace git2k8s10min
cat <<EOF | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: git2k8s10min
  namespace: argocd
spec:
  destination:
    namespace: git2k8s10min
    server: https://kubernetes.default.svc
  project: default
  source:
    repoURL: $HTTPS_REPO_URL
    path: .
    targetRevision: master
EOF
argocd app sync git2k8s10min
argocd app get git2k8s10min
kubectl get -n git2k8s10min svc/frontend pods
kubectl port-forward -n git2k8s10min svc/frontend 18080:80 

UI: http://localhost:18080

# Step 5: Deploy again but in auto mode – 2 min
kubectl create namespace git2k8s10min-live
cat <<EOF | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: git2k8s10min-live
  namespace: argocd
spec:
  destination:
    namespace: git2k8s10min-live
    server: https://kubernetes.default.svc  
  project: default
    source:
    repoURL: $HTTPS_REPO_URL
    path: .
    targetRevision: live
  syncPolicy:
    automated: {}
EOF

git checkout live
git merge master
git push –set-upstream origin live
