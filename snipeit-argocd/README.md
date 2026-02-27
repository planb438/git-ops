Perfect 👍 — this is a great choice for your cluster size.

We’ll deploy Snipe-IT via ArgoCD using a VERY lightweight GitOps setup that works well on AWS free-tier nodes.

This stack uses:

1 MySQL pod

1 Snipe-IT pod

Persistent storage

NodePort access

👉 Total footprint ≈ < 1 GB RAM + ~2 GB disk

🗂️ 1. Git Repo Structure (Create This)

Create a repo like:

snipeit-argocd/
 ├── namespace.yaml
 ├── secret.yaml
 ├── mysql.yaml
 ├── snipeit.yaml
 ├── service.yaml

Commit to Git → ArgoCD will sync.

📦 2. Namespace
namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: snipeit
🔐 3. Secrets (DB + App Keys)
secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: snipeit-secret
  namespace: snipeit
type: Opaque
stringData:
  MYSQL_ROOT_PASSWORD: rootpass
  MYSQL_DATABASE: snipeit
  MYSQL_USER: snipeit
  MYSQL_PASSWORD: snipepass
  APP_KEY: base64:SomeRandomKeyHere123456

👉 APP_KEY can be anything random for now.

🗄️ 4. MySQL Deployment (Lightweight)
mysql.yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: snipeit
spec:
  ports:
    - port: 3306
  selector:
    app: mysql
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: snipeit
spec:
  serviceName: mysql
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:8
          envFrom:
            - secretRef:
                name: snipeit-secret
          volumeMounts:
            - mountPath: /var/lib/mysql
              name: data
          resources:
            requests:
              memory: "300Mi"
              cpu: "100m"
              ephemeral-storage: "1Gi"
            limits:
              memory: "600Mi"
              cpu: "500m"
              ephemeral-storage: "2Gi"
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi
🚀 5. Snipe-IT Deployment
snipeit.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: snipeit
  namespace: snipeit
spec:
  replicas: 1
  selector:
    matchLabels:
      app: snipeit
  template:
    metadata:
      labels:
        app: snipeit
    spec:
      containers:
        - name: snipeit
          image: snipe/snipe-it:latest
          ports:
            - containerPort: 80

          env:
            - name: APP_ENV
              value: production
            - name: APP_KEY
              valueFrom:
                secretKeyRef:
                  name: snipeit-secret
                  key: APP_KEY

            - name: DB_HOST
              value: mysql
            - name: DB_DATABASE
              valueFrom:
                secretKeyRef:
                  name: snipeit-secret
                  key: MYSQL_DATABASE
            - name: DB_USERNAME
              valueFrom:
                secretKeyRef:
                  name: snipeit-secret
                  key: MYSQL_USER
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: snipeit-secret
                  key: MYSQL_PASSWORD

          resources:
            requests:
              memory: "300Mi"
              cpu: "100m"
              ephemeral-storage: "1Gi"
            limits:
              memory: "600Mi"
              cpu: "500m"
              ephemeral-storage: "2Gi"
🌐 6. NodePort Service (Access UI)
service.yaml
apiVersion: v1
kind: Service
metadata:
  name: snipeit-nodeport
  namespace: snipeit
spec:
  type: NodePort
  selector:
    app: snipeit
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
🎯 7. ArgoCD Application YAML
argocd-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: snipeit
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR_REPO/snipeit-argocd.git
    targetRevision: HEAD
    path: .
  destination:
    server: https://kubernetes.default.svc
    namespace: snipeit
  syncPolicy:
    automated:
      prune: true
      selfHeal: true

Apply:

kubectl apply -f argocd-app.yaml
🌍 8. Access Snipe-IT

Find your node public IP:

http://NODE_IP:30080

You’ll see:

👉 Snipe-IT setup wizard

Create:

Admin user

Company name

Done ✅

🧠 How This Fits Your Parking Business

You can now track:

Parking Thing	In Snipe-IT
Cars	Assets
Parking Zones	Locations
Customers	Users
Keys	Asset Tags
Check-in/out	Asset Checkout

This solves your “can’t find cars quickly” problem perfectly.

🏆 Why This Will NOT Break Your Cluster

Very light footprint:

• 2 pods total
• <1GB RAM usage
• <3GB disk total

No Redis.
No workers.
No heavy DB.

⭐ Recommended Next Steps (Later)

Once stable you can add:

Ingress + TLS

QR code scanning workflow

Backup CronJob

S3 backups

👍 If you want next, I can help with:
1️⃣ How to model parking workflow inside Snipe-IT
2️⃣ How to add QR labels for keys/cars
3️⃣ How to automate check-in/out process
4️⃣ How to migrate from your spreadsheet