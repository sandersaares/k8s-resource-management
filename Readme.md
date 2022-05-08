# Prerequisites

1. Azure Kubernetes Service cluster.
    * Add a user nodepool with 3 Linux nodes, D16as v5 SKU,
    * Azure CNI networking mode.
1. Deploy Prometheus and Node Exporter onto the cluster for data collection purposes.
    * `kubectl create namespace monitoring`
    * `kubectl apply -f infra/node-exporter.yml`
    * `kubectl apply -f infra/prometheus.yml`
1. Identify the external IP address assigned to the Prometheus service. Use this to visualize monitoring data.
    * `kubectl get services --namespace monitoring`
    * Open `http://<external IP>:9090` in browser to query metrics.

# Problem: CPU theft

Description: is it possible for a container to be given less CPU time than its request?

1. `kubectl apply -f cpu-theft/guaranteed.yml` - with guaranteed QoS, consumes 8 out of 16 cores.
1. Execute the query `irate(process_cpu_seconds_total{job="k8spods"}[60s])` in the Prometheus web GUI and keep observing it throughout the experiment. This measures CPU consumption in cores.
1. `kubectl apply -f cpu-theft/thief.yml` - burstable QoS, consumes 7 out of 16 cores.
1. `kubectl apply -f cpu-theft/victim.yml` - burstable QoS, consumes 2 out of 16 cores.

Result: all OK, CPU request is guaranteed and "victim" is given CPU from "thief".

# Problem: memory theft

Description: is it possible for a container to be given less memory than its request, and for it to be OOMKilled because of this?

1. `kubectl apply -f cpu-theft/guaranteed.yml` - with guaranteed QoS, consumes 20-ish GB.
1. Execute the queries `dotnet_total_memory_bytes` and `node_memory_MemAvailable_bytes` in the Prometheus web GUI and keep observing it throughout the experiment. These measure managed memory consumption and free server RAM.
1. `kubectl apply -f cpu-theft/thief.yml` - burstable QoS, consumes 30-ish GB.
1. `kubectl apply -f cpu-theft/victim.yml` - burstable QoS, consumes 4 GB and then allocates another 6 GB.

Result: all OK, memory request is guaranteed, "thief" is OOMKilled when "victim" allocates additional memory.