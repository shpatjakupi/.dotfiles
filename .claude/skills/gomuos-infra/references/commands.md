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

### MySQL via SSH — Backtick workaround
The `order` table is a SQL reserved word. Backtick escaping breaks through SSH+kubectl layers.
Use `printf` with `\x60` (backtick) and pipe via a temp SQL file:
```bash
# Write query to file (using \x60 for backticks), copy to pod, execute
ssh -T root@46.224.215.213 'printf "SELECT * FROM \x60order\x60 ORDER BY Id DESC LIMIT 5;\n" > /tmp/q.sql && kubectl cp /tmp/q.sql gomuos/mysql-0:/tmp/q.sql && kubectl exec -n gomuos statefulset/mysql -- bash -c "mysql -u admin -pJordguld1!123412341234 ordrupspizza < /tmp/q.sql"'
```

### Useful queries
```sql
-- Recent orders with payment status
SELECT Id, Payment_id, Ordered_date, Is_payment_successful, Order_done FROM `order` ORDER BY Id DESC LIMIT 10;

-- Check blocked IPs
SELECT * FROM blocked_ips;

-- Clear a blocked IP
DELETE FROM blocked_ips WHERE ip_address = '1.2.3.4';

-- Check admin users
SELECT id, username, email FROM admin;

-- DB timezone check
SELECT @@global.time_zone, @@session.time_zone, NOW(), UTC_TIMESTAMP();
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

## Troubleshooting Checklist

### Orders not showing in admin dashboard
1. Check `Is_payment_successful` on the order — if `0`, the Bambora callback failed
2. Check frontend logs for callback errors: `kubectl logs -l app=ordrupspizza-frontend -n gomuos --tail=50`
3. Check backend logs for callback receipt: `kubectl logs -l app=ordrupspizza-backend -n gomuos --tail=50 | grep callback`
4. Manually mark paid if needed: `UPDATE \`order\` SET Is_payment_successful = 1 WHERE Id = <id>;`

### 400 Bad Request errors (browser-specific)
- Firefox sends `DNT` and `TE` headers — must be in `StrictHttpFirewall` allowed list (`Config.java`)
- Check backend logs for rejected headers

### CORS / 403 errors
- Check `APP_DOMAIN` env var in backend secret — must be `ordrupspizza` (no `.dk`)
- Check if client IP got auto-blocked: `SELECT * FROM blocked_ips;`
- K8s internal IPs (`10.42.*`) are whitelisted in `CorsFilter`

### WebSocket "Connection closed"
- SockJS tries native WS first, falls back to XHR streaming — "Connection closed" on first transport is normal
- Check CSP `connect-src` includes `wss://ordrupspizza.dk`
- Check ingress has `/ws` path routing to backend
- STOMP `CONNECTED` message in console = working

### Frontend env vars empty in production
- `NEXT_PUBLIC_*` vars are baked at **build time**, not runtime
- Must be passed as `build-args` in `.github/workflows/build.yml`
- After changing build args, need new Docker build + pod restart
