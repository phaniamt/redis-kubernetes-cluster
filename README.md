# redis-kubernetes-cluster
### Create redis config ###
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: redis-cluster
      namespace: default
    data:
      update-node.sh: |
        #!/bin/sh
        REDIS_NODES="/data/nodes.conf"
        sed -i -e "/myself/ s/[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}/${POD_IP}/" ${REDIS_NODES}
        exec "$@"
      redis.conf: |+
        cluster-enabled yes
        cluster-require-full-coverage no
        cluster-node-timeout 15000
        cluster-config-file /data/nodes.conf
        cluster-migration-barrier 1
        appendonly yes
        protected-mode no
### Create redis-cluster service ###
    apiVersion: v1
    kind: Service
    metadata:
      name: redis-cluster
      namespace: beta
    spec:
      type: ClusterIP
      ports:
      - port: 6379
        targetPort: 6379
        name: client
      - port: 16379
        targetPort: 16379
        name: gossip
      selector:
        app: redis-cluster
