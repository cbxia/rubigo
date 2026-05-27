
flowchart LR
  %% ====== CAs ======
  subgraph CAs[Certificate Authorities]
    KCA["Kubernetes CA\n/etc/kubernetes/pki/ca.crt"]
    ECA["etcd CA\n/etc/kubernetes/pki/etcd/ca.crt"]
    FPCA["Front-proxy CA\n/etc/kubernetes/pki/front-proxy-ca.crt"]
  end

  %% ====== External / Users ======
  subgraph Clients[Clients]
    kubectl["kubectl / admin.conf\n(client cert embedded)"]
  end

  %% ====== Control Plane ======
  subgraph CP[Control Plane]
    apiserver["kube-apiserver\n:6443\nserving cert: apiserver.crt"]
    cm["kube-controller-manager\ncontroller-manager.conf\n(client cert embedded)"]
    sched["kube-scheduler\nscheduler.conf\n(client cert embedded)"]
  end

  %% ====== Nodes ======
  subgraph Nodes[Nodes]
    kubelet["kubelet\n:10250\nserving cert: kubelet server cert\nclient cert: kubelet client cert"]
  end

  %% ====== etcd ======
  subgraph ETCD["etcd (stacked)"]
    etcd1["etcd member\nclient port :2379\nserver.crt"]
    etcd2["etcd member\npeer port :2380\npeer.crt"]
  end

  %% ====== Aggregation (optional but common) ======
  subgraph Agg["Aggregation Layer (optional)"]
    aggapi["Aggregated API (e.g. metrics-server)\nverifies front-proxy client cert"]
  end

  %% ====== Flows ======
