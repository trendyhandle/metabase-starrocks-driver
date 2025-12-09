# Metabase StarRocks Driver

A custom Metabase driver for [StarRocks](https://www.starrocks.io/) that fixes MySQL protocol compatibility issues and adds proper multi-catalog support.

## Why This Driver?

Metabase's built-in MySQL driver doesn't work properly with StarRocks because:

1. **`SHOW GRANTS FOR CURRENT_USER` is unsupported** - StarRocks uses a different privilege system than MySQL, causing metadata sync to fail with:
   ```
   No viable statement for input 'SHOW GRANTS FOR CURRENT_USER'
   ```

2. **Multi-catalog support** - StarRocks external catalogs (Hive, Iceberg, etc.) require `catalog.database` format which the MySQL driver doesn't handle well.

This driver extends Metabase's `sql-jdbc` driver directly, bypassing the problematic MySQL-specific code while maintaining full query compatibility.

## Requirements

- **StarRocks**: v3.2+ (for external catalog support)
- **Java**: JDK 11-21 (for building)
- **Clojure CLI**: For building the JAR

## Building

```bash
# Install Clojure CLI (macOS)
brew install clojure/tools/clojure

# Build the driver JAR
cd metabase-starrocks-driver
clojure -T:build uber

# Output: target/starrocks.metabase-driver.jar
```

## Deployment (EKS)

After merging changes, deploy to your Metabase instance:

```bash
# 1. Copy built JAR to Metabase pod's plugin storage
POD=$(kubectl get pods -n metabase-dev -o jsonpath='{.items[0].metadata.name}')
kubectl cp target/starrocks.metabase-driver.jar metabase-dev/$POD:/metabase-data/plugins/

# 2. Restart Metabase (scale down then up to avoid H2 lock issues)
kubectl scale deployment metabase -n metabase-dev --replicas=0
kubectl scale deployment metabase -n metabase-dev --replicas=1
```

> **Note**: The deployment uses an init container that copies plugins from `/metabase-data/plugins/` to an emptyDir volume at startup. This allows rolling updates to work with RWO volumes.

## Configuration in Metabase

1. Go to **Admin → Databases → Add Database**
2. Select **"StarRocks"**
3. Configure:

| Field | Description |
|-------|-------------|
| Host | StarRocks FE hostname |
| Port | MySQL protocol port |
| Catalog | StarRocks catalog name |
| Database | Database within catalog (optional) |
| Username | StarRocks user |
| Password | User password |

### Catalog Examples

- **Internal catalog**: `default_catalog`
- **Hive catalog**: `certified`
- **Iceberg/Polaris catalog**: `bulk`

Leave **Database** empty to see all databases in the catalog.

## Project Structure

```
metabase-starrocks-driver/
├── deps.edn                         # Dependencies & build config
├── build.clj                        # Build script
├── src/metabase/driver/
│   └── starrocks.clj                # Driver implementation
└── resources/
    └── metabase-plugin.yaml         # Plugin manifest
```
