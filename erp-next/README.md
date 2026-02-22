📦 Argo CD Application for ERPNext

🚀 Deploy ERPNext
bash
# Apply the Argo CD application
kubectl apply -f erpnext-app.yaml -n argocd

# Monitor the deployment
argocd app get erpnext -w

# Check pods (will take 5-10 minutes to initialize)
kubectl get pods -n erpnext -w
🔧 What Gets Deployed
The Helm chart creates multiple components :

Gunicorn - Python web server (main ERPNext app)

Nginx - Static file serving

Workers - Background job processing (default, short, long queues)

Scheduler - Automated tasks

SocketIO - Real-time communications

Create-site job - Initial site setup

📊 Access ERPNext
bash
# Get the NodePort
kubectl get svc -n erpnext erpnext-nginx -o jsonpath='{.spec.ports[0].nodePort}'

# Access via any worker node
http://<your-worker-node-ip>:32000
Default login:

Username: Administrator

Password: Generated during site creation (check logs)

bash
# Get admin password from logs
kubectl logs -n erpnext -l job-name=erpnext-create-site --tail=50 | grep "Administrator"
🛠️ Custom Apps (HR, Manufacturing, etc.)
If you need additional apps like HRMS, you'll need a custom Docker image :

yaml
# Modified values section
values: |
  image:
    repository: your-dockerhub/erpnext-custom  # Your custom image
    tag: with-hrms
  
  createSite:
    enabled: true
    siteName: "erpnext.local"
    installApps:
      - hrms  # Install HR app after site creation
Build custom image:

bash
# Create apps.json
cat > apps.json <<EOF
[
  {"url": "https://github.com/frappe/erpnext", "branch": "version-15"},
  {"url": "https://github.com/frappe/hrms", "branch": "version-15"}
]
EOF

export APPS_JSON_BASE64=$(base64 -w 0 apps.json)

docker build \
  --build-arg=FRAPPE_PATH=https://github.com/frappe/frappe \
  --build-arg=FRAPPE_BRANCH=version-15 \
  --build-arg=APPS_JSON_BASE64=$APPS_JSON_BASE64 \
  --tag=yourdockerhub/erpnext:custom \
  -f Dockerfile .
⚠️ Important Notes for Your Setup
RWX Storage: ERPNext needs ReadWriteMany for the sites volume . Your local-path provisioner only supports ReadWriteOnce. Options:

Quick fix: Deploy NFS provisioner (as shown in the helm docs) 

Alternative: Use single node for now (RWO works if pods schedule on same node)

Memory Requirements: ERPNext needs at least 4GB total . Your t3.medium (4GB) is borderline but should work for testing.

Persistent Volumes: The chart creates multiple PVCs :

Main sites volume (10GB)

Logs volume (2GB)

Database: By default, the chart deploys MariaDB internally. If you want to use an external database (recommended for production), set mariadbHost .

🏁 Troubleshooting
bash
# Check site creation job
kubectl logs -n erpnext -l job-name=erpnext-create-site

# Check worker logs
kubectl logs -n erpnext -l app.kubernetes.io/component=worker-d

# If pods are pending due to storage
kubectl describe pvc -n erpnext

# Fix storage by deploying NFS provisioner [citation:2]
kubectl create namespace nfs
helm repo add nfs-ganesha-server-and-external-provisioner https://kubernetes-sigs.github.io/nfs-ganesha-server-and-external-provisioner
helm upgrade --install -n nfs in-cluster nfs-ganesha-server-and-external-provisioner/nfs-server-provisioner \
  --set 'storageClass.mountOptions={vers=4.1}' \
  --set persistence.enabled=true \
  --set persistence.size=8Gi
This gets you a fully GitOps-managed ERPNext deployment. Once running, you can configure the parking module just like we discussed for Nextcloud, but with full ERP capabilities for the other businesses.

🔧 Use the Correct Raw URL
bash
# CORRECT - Use raw.githubusercontent.com
kubectl apply -f https://raw.githubusercontent.com/planb438/ERPNext/main/erpnext-app.yaml

Option 2: Apply Locally (Quickest)
bash
# Save the fixed YAML above to a local file
nano erpnext-app-fixed.yaml
# Paste the corrected content

# Apply locally
kubectl apply -f erpnext-app-fixed.yaml -n argocd
Option 3: Use Curl to Download Raw
bash
curl -LO https://raw.githubusercontent.com/planb438/ERPNext/main/erpnext-app.yaml
# Edit to fix indentation
kubectl apply -f erpnext-app.yaml -n argocd
✅ After Successful Apply
bash
# Check application status
argocd app get erpnext

# Watch pods come up
kubectl get pods -n erpnext -w
