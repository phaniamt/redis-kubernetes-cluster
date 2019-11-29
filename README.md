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
      namespace: default
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
### Deploy redis statefulset ###
    apiVersion: apps/v1
    kind: StatefulSet
    metadata:
      name: redis-cluster
      namespace: deafult
    spec:
      serviceName: redis-cluster
      replicas: 6
      selector:
        matchLabels:
          app: redis-cluster
      template:
        metadata:
          labels:
            app: redis-cluster
        spec:
          nodeSelector:
            server: beta-db
          containers:
          - name: redis
            image: redis:5.0.7-alpine
            ports:
            - containerPort: 6379
              name: client
            - containerPort: 16379
              name: gossip
            command: ["/conf/update-node.sh", "redis-server", "/conf/redis.conf"]
            env:
            - name: POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            readinessProbe:
              exec:
                command:
                - sh
                - -c
                - "redis-cli -h $(hostname) ping"
              initialDelaySeconds: 15
              timeoutSeconds: 5
            livenessProbe:
              exec:
                command:
                - sh
                - -c
                - "redis-cli -h $(hostname) ping"
              initialDelaySeconds: 20
              periodSeconds: 3
            volumeMounts:
            - name: conf
              mountPath: /conf
              readOnly: false
            - name: redis-data
              mountPath: /data
              readOnly: false
          volumes:
          - name: conf
            configMap:
              name: redis-cluster
              defaultMode: 0755
      volumeClaimTemplates:
      - metadata:
          name: redis-data
        spec:
          accessModes: [ "ReadWriteOnce" ]
          storageClassName: standard
          resources:
            requests:
              storage: 10Gi
### Create redis cluster ###
    kubectl exec -it redis-cluster-0 -- redis-cli --cluster create --cluster-replicas 1 $(kubectl get pods -l app=redis-cluster -o jsonpath='{range.items[*]}{.status.podIP}:6379 ')
