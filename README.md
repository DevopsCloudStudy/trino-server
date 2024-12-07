# Blog: for trino https://github.com/alexcpn/presto_in_kubernetes
# trino-server

pg3391 
#pg3391
@pg3391 
Pankaj Gupta's Blog Docqs.in
``` bash
#!/bin/bash

# Variables
NAMESPACE="sandbox"
TRINO_URL="http://test-trino.example.com"
TRINO_CLI="/usr/bin/trino"  # Update path if CLI is located elsewhere
TRINO_USER="{{vaultkv_secret.trino_user}}"  # Replace with actual user or env variable
TRINO_PASSWORD="{{vaultkv_secret.trino_cli_psswd}}"  # Replace with actual password or env variable
EXPECTED_TPCH_COUNT=1500000
EXPECTED_TPCDS_COUNT=100000

# Function to check command execution success
check_status() {
  if [ $? -ne 0 ]; then
    echo "Error: $1 failed!"
    exit 1
  fi
}

# 1. Health Check for Trino Endpoint
echo "Checking Trino endpoint health..."
HEALTH_STATUS=$(curl -s -o /dev/null -w "%{http_code}" "$TRINO_URL/v1/status")
if [ "$HEALTH_STATUS" -ne 200 ]; then
  echo "Error: Trino endpoint is not healthy! Received HTTP status: $HEALTH_STATUS"
  exit 1
fi
echo "Trino endpoint is healthy (HTTP $HEALTH_STATUS)."

# 2. Check Pod Status in Kubernetes
echo "Checking pod statuses in namespace '$NAMESPACE'..."
PODS_STATUS=$(kubectl get pods -n "$NAMESPACE" --field-selector=status.phase!=Running -o jsonpath='{.items[*].metadata.name}')
if [ -n "$PODS_STATUS" ]; then
  echo "Error: The following pods are not in 'Running' state:"
  echo "$PODS_STATUS"
  exit 1
fi
echo "All pods in namespace '$NAMESPACE' are running."

# 3. Run Basic TPC-H Query
echo "Running basic TPC-H query..."
export TRINO_PASSWORD="$TRINO_PASSWORD"
TPCH_QUERY_RESULT=$("$TRINO_CLI" --server "$TRINO_URL" --user "$TRINO_USER" --password --execute 'SELECT COUNT(*) FROM tpch.sf1.orders')
check_status "TPC-H Query"
echo "TPC-H Query Result: $TPCH_QUERY_RESULT"

if [[ "$TPCH_QUERY_RESULT" != *"$EXPECTED_TPCH_COUNT"* ]]; then
  echo "Error: TPC-H query result validation failed!"
  exit 1
fi

# 4. Check Trino Worker Status
echo "Checking Trino worker status..."
WORKER_STATUS=$("$TRINO_CLI" --server "$TRINO_URL" --user "$TRINO_USER" --password --execute 'SELECT * FROM system.runtime.nodes WHERE coordinator = false')
check_status "Check Trino Workers"
echo "Trino Worker Status:"
echo "$WORKER_STATUS"

if [[ "$WORKER_STATUS" != *"active"* ]]; then
  echo "Error: No active workers found!"
  exit 1
fi

# 5. Run Basic TPC-DS Query
echo "Running basic TPC-DS query..."
TPCDS_QUERY_RESULT=$("$TRINO_CLI" --server "$TRINO_URL" --user "$TRINO_USER" --password --execute 'SELECT COUNT(*) FROM tpcds.sf1.customer')
check_status "TPC-DS Query"
echo "TPC-DS Query Result: $TPCDS_QUERY_RESULT"

if [[ "$TPCDS_QUERY_RESULT" != *"$EXPECTED_TPCDS_COUNT"* ]]; then
  echo "Error: TPC-DS query result validation failed!"
  exit 1
fi

echo "All checks completed successfully!"

```
