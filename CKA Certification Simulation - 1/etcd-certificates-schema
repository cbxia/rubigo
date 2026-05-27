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
  kubectl -->|HTTPS :6443\nClient presents: admin cert\nServer presents: apiserver.crt\nAPIServer trusts client via KCA| apiserver

  cm -->|HTTPS :6443\nClient presents: controller-manager cert\nServer presents: apiserver.crt\nAPIServer trusts client via KCA| apiserver
  sched -->|HTTPS :6443\nClient presents: scheduler cert\nServer presents: apiserver.crt\nAPIServer trusts client via KCA| apiserver

  kubelet -->|HTTPS :6443\nClient presents: kubelet client cert\nAPIServer trusts via KCA| apiserver
  apiserver -->|HTTPS :10250\nClient presents: apiserver-kubelet-client.crt\nKubelet trusts via KCA| kubelet

  apiserver -->|HTTPS :2379\nClient presents: apiserver-etcd-client.crt\netcd trusts via ECA\netcd presents server.crt| etcd1

  etcd1 <--> |HTTPS :2380\nMutual TLS\npeer cert used for both\nlistening + dialing peers| etcd2

  apiserver -->|HTTPS to aggregated API\nClient presents: front-proxy-client.crt\nAgg API trusts via FPCA| aggapi
