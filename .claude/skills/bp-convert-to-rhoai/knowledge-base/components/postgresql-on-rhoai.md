---
name: postgresql-on-rhoai
description: PostgreSQL database deployment on RHOAI using Bitnami image with nullable security contexts
metadata:
  type: component
components: [postgresql, postgres, database]
deployment_types: [helm, oc-apply]
resource_types: [storage, security-context]
architecture: []
source_examples:
  - blueprint: "aiq"
    source_repo: "https://github.com/NVIDIA-AI-Blueprints/aiq"
    fork_repo: "https://github.com/rh-ai-quickstart/nvidia-aiq"
    notes: "Standard Bitnami PostgreSQL with overlay strategy security context handling"
    approach: "A"
---

# PostgreSQL on RHOAI

## Overview

PostgreSQL is commonly used in NVIDIA blueprints as a persistent database for job state, checkpoints, agent memory, and application data. The Bitnami PostgreSQL image works well on RHOAI with proper security context configuration.

## Conversion Pattern

### Standard Deployment Configuration

PostgreSQL typically requires:
- **Persistent storage**: For database data
- **Security context**: Must be set to null for RHOAI to allow SCC management
- **Init containers**: For database initialization (create databases, grant permissions)
- **Health checks**: For readiness and liveness probes

### RHOAI-Specific Modifications

#### 1. Security Context

**Standard Kubernetes** (original blueprint):
```yaml
apps:
  postgres:
    podSecurityContext:
      runAsNonRoot: true
      fsGroup: 1001
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop:
          - ALL
```

**RHOAI** (set to null in overlay):
```yaml
apps:
  postgres:
    podSecurityContext: null  # Let OpenShift SCC manage
    securityContext: null
```

**Why?** OpenShift's restricted-v2 SCC automatically assigns appropriate UIDs and GIDs from the namespace's UID/GID range. Setting these explicitly causes conflicts.

#### 2. Persistent Storage

**Example PVC configuration**:
```yaml
persistence:
  - name: postgres-data
    accessModes:
      - ReadWriteOnce
    size: 10Gi
    # storageClass: "" # Use default or set explicitly for RHOAI

volumes:
  - name: postgres-data
    persistentVolumeClaim:
      claimName: postgres-data

volumeMounts:
  - name: postgres-data
    mountPath: /var/lib/postgresql/data
```

**RHOAI considerations**:
- Use default storage class or specify one available in your cluster
- ReadWriteOnce is sufficient for single-replica PostgreSQL
- Size depends on application needs (10Gi is typical for job/checkpoint storage)

#### 3. Database Initialization

Blueprints often need to initialize databases and permissions. Use an init container in the application pod (not in PostgreSQL pod):

**Example from aiq blueprint**:
```yaml
initContainers:
  - name: db-init
    image: bitnami/postgresql:latest
    command:
      - sh
      - -c
      - |
        echo "Waiting for postgres..."
        until pg_isready -h postgres-service -U $(DB_USER_NAME) -d app_db; do
          sleep 2
        done
        echo "Running init script..."
        psql -h postgres-service -U $(DB_USER_NAME) -d app_db -f /db-init/init.sql
    env:
      - name: PGPASSWORD
        valueFrom:
          secretKeyRef:
            name: app-credentials
            key: DB_USER_PASSWORD
      - name: DB_USER_NAME
        valueFrom:
          secretKeyRef:
            name: app-credentials
            key: DB_USER_NAME
    volumeMounts:
      - name: postgres-init
        mountPath: /db-init
        readOnly: true
```

**ConfigMap for init script**:
```yaml
configMaps:
  - name: postgres-init
    data:
      init.sql: |
        CREATE DATABASE IF NOT EXISTS app_checkpoints;
        GRANT ALL PRIVILEGES ON DATABASE app_db TO ${DB_USER_NAME};
        GRANT ALL PRIVILEGES ON DATABASE app_checkpoints TO ${DB_USER_NAME};
        
        CREATE TABLE IF NOT EXISTS job_info (
          job_id VARCHAR PRIMARY KEY,
          status VARCHAR NOT NULL,
          created_at TIMESTAMP WITH TIME ZONE,
          updated_at TIMESTAMP WITH TIME ZONE
        );
```

#### 4. Environment Variables and Secrets

**PostgreSQL pod configuration**:
```yaml
env:
  POSTGRES_DB: app_db

secretEnv:
  POSTGRES_USER: DB_USER_NAME  # Maps to secret key
  POSTGRES_PASSWORD: DB_USER_PASSWORD  # Maps to secret key
```

**Application connection string**:
```yaml
env:
  DATABASE_URL: postgresql://$(DB_USER_NAME):$(DB_USER_PASSWORD)@postgres-service:5432/app_db
  # Or separate components
  DB_HOST: postgres-service
  DB_PORT: "5432"
  DB_NAME: app_db
```

**Secret creation** (OpenShift overlay approach):
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-credentials
type: Opaque
stringData:
  DB_USER_NAME: app_user
  DB_USER_PASSWORD: changeme  # Change in production
```

#### 5. Health Checks

**Liveness probe**:
```yaml
livenessProbe:
  exec:
    command:
      - sh
      - -c
      - pg_isready -U $POSTGRES_USER -d $POSTGRES_DB
  initialDelaySeconds: 10
  periodSeconds: 5
  timeoutSeconds: 5
  failureThreshold: 5
```

**Readiness probe**:
```yaml
readinessProbe:
  exec:
    command:
      - sh
      - -c
      - pg_isready -U $POSTGRES_USER -d $POSTGRES_DB
  initialDelaySeconds: 5
  periodSeconds: 5
  timeoutSeconds: 5
  failureThreshold: 5
```

#### 6. Service Configuration

**Standard ClusterIP service**:
```yaml
service:
  type: ClusterIP
  ports:
    - name: postgres
      port: 5432
      targetPort: 5432
```

No Route needed - PostgreSQL is internal-only.

### Complete Example (Helm Values)

**Base values.yaml**:
```yaml
apps:
  postgres:
    enabled: true
    replicas: 1
    image:
      repository: bitnami/postgresql
      tag: latest
      pullPolicy: IfNotPresent
    
    service:
      type: ClusterIP
      ports:
        - name: postgres
          port: 5432
          targetPort: 5432
    
    ports:
      - name: postgres
        containerPort: 5432
        protocol: TCP
    
    secretEnv:
      POSTGRES_USER: DB_USER_NAME
      POSTGRES_PASSWORD: DB_USER_PASSWORD
    
    env:
      POSTGRES_DB: app_db
    
    persistence:
      - name: postgres-data
        accessModes:
          - ReadWriteOnce
        size: 10Gi
    
    volumes:
      - name: postgres-data
        persistentVolumeClaim:
          claimName: postgres-data
    
    volumeMounts:
      - name: postgres-data
        mountPath: /var/lib/postgresql/data
    
    healthCheck:
      enabled: true
      livenessProbe:
        exec:
          command:
            - sh
            - -c
            - pg_isready -U $POSTGRES_USER -d $POSTGRES_DB
        initialDelaySeconds: 10
        periodSeconds: 5
        timeoutSeconds: 5
        failureThreshold: 5
      readinessProbe:
        exec:
          command:
            - sh
            - -c
            - pg_isready -U $POSTGRES_USER -d $POSTGRES_DB
        initialDelaySeconds: 5
        periodSeconds: 5
        timeoutSeconds: 5
        failureThreshold: 5
    
    podSecurityContext: null  # Will be set to null in OpenShift overlay
```

**OpenShift overlay (values-openshift.yaml)**:
```yaml
apps:
  postgres:
    podSecurityContext: null
    securityContext: null
```

## Known Issues and Gotchas

### Issue: PostgreSQL fails to start with permission denied errors

**Symptoms**: Pod logs show "permission denied" when trying to write to `/var/lib/postgresql/data`

**Cause**: Security context conflicts with OpenShift SCC's automatically assigned UIDs

**Solution**: Ensure `podSecurityContext: null` and `securityContext: null` are set in the OpenShift overlay. Do NOT set `fsGroup` or `runAsUser` explicitly.

### Issue: Init scripts fail with "psql: connection refused"

**Symptoms**: Init container fails before PostgreSQL is ready

**Cause**: PostgreSQL takes time to start; init container tries to connect too early

**Solution**: Use `pg_isready` in a retry loop:
```bash
until pg_isready -h postgres-service -U $(DB_USER_NAME) -d app_db; do
  echo "Waiting for PostgreSQL..."
  sleep 2
done
```

### Issue: Database credentials not found

**Symptoms**: `POSTGRES_USER` or `POSTGRES_PASSWORD` environment variables are empty

**Cause**: Secret not created or mounted incorrectly

**Solution**: 
1. Ensure secret exists: `oc get secret app-credentials -n namespace`
2. Check secret has correct keys: `oc describe secret app-credentials -n namespace`
3. Verify `secretEnv` mapping in values matches secret keys

### Issue: PVC stuck in Pending state

**Symptoms**: PostgreSQL pod can't start because PVC is Pending

**Cause**: No storage class available or insufficient resources

**Solution**:
```bash
# Check available storage classes
oc get sc

# Check PVC status
oc describe pvc postgres-data -n namespace

# If no default storage class, set one explicitly
apps:
  postgres:
    persistence:
      - name: postgres-data
        storageClassName: gp3-csi  # Use available storage class
```

## Dependencies

PostgreSQL typically has no dependencies on other components, but other components depend on it:
- Backend applications (connect to PostgreSQL)
- Init containers (initialize databases)

Ensure PostgreSQL is **ready** before dependent applications try to connect. Use readiness probes and init container retry loops.

## Testing Notes

### Verify PostgreSQL is Running

```bash
# Check pod status
oc get pods -l app=postgres -n namespace

# Check logs
oc logs -l app=postgres -n namespace

# Check PVC
oc get pvc postgres-data -n namespace
```

### Test Database Connection

```bash
# From within the cluster (exec into application pod)
psql -h postgres-service -U app_user -d app_db -c "SELECT version();"

# Or from PostgreSQL pod
oc exec -it postgres-pod -n namespace -- psql -U app_user -d app_db -c "SELECT version();"
```

### Verify Credentials Secret

```bash
# Check secret exists
oc get secret app-credentials -n namespace

# View secret keys (not values)
oc describe secret app-credentials -n namespace

# Decode secret value (for debugging)
oc get secret app-credentials -n namespace -o jsonpath='{.data.DB_USER_NAME}' | base64 -d
```

## Alternative Approaches

### Using PostgreSQL Operator

For production deployments, consider using a PostgreSQL operator like:
- **Crunchy PostgreSQL Operator**: Enterprise-grade PostgreSQL management
- **Zalando PostgreSQL Operator**: Kubernetes-native PostgreSQL clusters

These provide:
- High availability
- Automated backups
- Monitoring
- Connection pooling

### Using Managed PostgreSQL

For cloud deployments, consider managed PostgreSQL services:
- AWS RDS
- Google Cloud SQL
- Azure Database for PostgreSQL
- Red Hat OpenShift Database Access

These eliminate infrastructure management but require network connectivity configuration.

## Related Patterns

- [[security-contexts-scc]] - Security context requirements
- [[rhoai-pvc-initialization]] - PVC initialization patterns
- [[helm-openshift-conditionals]] - Helm overlay strategy for OpenShift
