# Common GomuOS Commands

## kubectl - Check Status
```bash
# All pods in gomuos namespace
kubectl get pods -n gomuos

# All services
kubectl get svc -n gomuos

# ArgoCD apps
kubectl get applications -n argocd

# Ingress + certificates
kubectl get ingress,certificate -n gomuos

# Pod logs
kubectl logs -l app=ordrupspizza-backend -n gomuos --tail=50
kubectl logs -l app=ordrupspizza-frontend -n gomuos --tail=50
```

## kubectl - Deploy New Image
```bash
# Restart pods to pull latest :latest image
kubectl rollout restart deployment/ordrupspizza-frontend -n gomuos
kubectl rollout restart deployment/ordrupspizza-backend -n gomuos

# Watch rollout
kubectl rollout status deployment/ordrupspizza-backend -n gomuos
```

## kubectl - MySQL
```bash
# Connect to MySQL
kubectl exec -i mysql-0 -n gomuos -- mysql -u root -p'<mysql-root-password>'

# Run a query
kubectl exec mysql-0 -n gomuos -- mysql -u root -p'<mysql-root-password>' -e "SHOW DATABASES;" 2>/dev/null

# Import SQL dump
kubectl exec -i mysql-0 -n gomuos -- mysql -u root -p'<mysql-root-password>' <dbname> 2>/dev/null < /root/dump.sql
```

## MySQL - Refresh DB from AWS
```bash
# 1. Export from AWS RDS (run on server)
mysqldump -h database-orderapp.cy9tfut8dd6j.eu-north-1.rds.amazonaws.com \
  -u admin -p'<mysql-root-password>' --set-gtid-purged=OFF <dbname> 2>/dev/null > /root/<dbname>_fresh.sql

# 2. Drop and recreate
kubectl exec mysql-0 -n gomuos -- mysql -u root -p'<mysql-root-password>' \
  -e "DROP DATABASE <dbname>; CREATE DATABASE <dbname>; GRANT ALL PRIVILEGES ON <dbname>.* TO 'admin'@'%'; FLUSH PRIVILEGES;" 2>/dev/null

# 3. Import
kubectl exec -i mysql-0 -n gomuos -- mysql -u root -p'<mysql-root-password>' <dbname> 2>/dev/null < /root/<dbname>_fresh.sql
```

## Apply Manifests Manually (bypass ArgoCD wait)
```bash
# Copy file to server and apply
scp <local-file.yaml> root@46.224.215.213:/root/
ssh root@46.224.215.213 "kubectl apply -f /root/<file.yaml>"
```

## ArgoCD - Manually Sync
```bash
# ArgoCD syncs automatically, but to force:
# Go to ArgoCD UI → click app → Sync
# Or delete pod to force redeploy with latest image
kubectl delete pod -l app=ordrupspizza-backend -n gomuos
```

## infra-gitops - Push and Auto-deploy
```bash
cd C:\Users\shpat\Desktop\projects\infra-gitops
git add -A && git commit -m "message" && git push
# ArgoCD picks up changes within ~3 minutes automatically
```

## SSL Certificate Status
```bash
kubectl get certificate -n gomuos
kubectl describe certificate ordrupspizza-tls -n gomuos
kubectl get challenges -n gomuos
```
