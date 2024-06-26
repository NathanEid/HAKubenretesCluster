# Kubernetes Cluster Setup Documentation

This documentation outlines the steps required to configure a Kubernetes cluster with 6 nodes: 3 control plane nodes and 3 worker nodes.

## Prerequisites

Ensure you are logged in as the root user on all nodes.

### Step 1: Install and Configure HAProxy on Control Plane Nodes

1. **Install HAProxy** on all three control plane nodes:
    ```sh
    apt install haproxy -y
    ```

2. **Edit the HAProxy configuration file** `/etc/haproxy/haproxy.cfg`:
    ```sh
    vim /etc/haproxy/haproxy.cfg
    ```

3. **Add the following configuration**:
    ```plaintext
    global
        log /dev/log    local0
        group haproxy
        daemon
        ca-base /etc/ssl/certs
        crt-base /etc/ssl/private
        ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384
        ssl-default-bind-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
        ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets

    defaults
        log     global
        mode    http
        option  httplog
        option  dontlognull
        timeout connect 5000
        timeout client  50000
        timeout server  50000
        timeout http-keep-alive 10s
        timeout check           10s
        errorfile 400 /etc/haproxy/errors/400.http
        errorfile 403 /etc/haproxy/errors/403.http
        errorfile 408 /etc/haproxy/errors/408.http
        errorfile 500 /etc/haproxy/errors/500.http
        errorfile 502 /etc/haproxy/errors/502.http
        errorfile 503 /etc/haproxy/errors/503.http
        errorfile 504 /etc/haproxy/errors/504.http

    frontend apiserver
        bind *:8443
        default_backend apiserverbackend

    backend apiserverbackend
        option httpchk GET /healthz
        http-check expect status 200
        balance     roundrobin
        server master01 192.168.80.100:6443 check
        server master02 192.168.80.101:6443 check
        server master03 192.168.80.102:6443 check
    ```

4. **Verify and restart HAProxy**:
    ```sh
    haproxy -f /etc/haproxy/haproxy.cfg -c
    systemctl restart haproxy
    ```

### Step 2: Install and Configure Keepalived on Control Plane Nodes

1. **Install Keepalived** on all three control plane nodes:
    ```sh
    apt install keepalived -y
    ```

2. **Edit the Keepalived configuration file** `/etc/keepalived/keepalived.conf`:

    - For `master01`:
        ```sh
        vim /etc/keepalived/keepalived.conf
        ```

        ```plaintext
        vrrp_script chk_haproxy {
            script "killall -0 haproxy"
            interval 2
            weight 2
        }

        vrrp_instance VI_1 {
            interface ens18
            state MASTER
            virtual_router_id 51
            priority 102
            advert_int 1
            authentication {
                auth_type PASS
                auth_pass Qwerty@1234
            }
            virtual_ipaddress {
                192.168.80.110
            }
            unicast_src_ip 192.168.80.100
            unicast_peer {
              192.168.80.101
              192.168.80.102
            }
            track_script {
                chk_haproxy
            }
        }
        ```

    - For `master02`:
        ```sh
        vim /etc/keepalived/keepalived.conf
        ```

        ```plaintext
        vrrp_script chk_haproxy {
            script "killall -0 haproxy"
            interval 2
            weight 2
        }

        vrrp_instance VI_1 {
            interface ens18
            state BACKUP
            virtual_router_id 51
            priority 101
            advert_int 1
            authentication {
                auth_type PASS
                auth_pass Qwerty@1234
            }
            virtual_ipaddress {
                192.168.80.110
            }
            unicast_src_ip 192.168.80.101
            unicast_peer {
              192.168.80.100
              192.168.80.102
            }
            track_script {
                chk_haproxy
            }
        }
        ```

    - For `master03`:
        ```sh
        vim /etc/keepalived/keepalived.conf
        ```

        ```plaintext
        vrrp_script chk_haproxy {
            script "killall -0 haproxy"
            interval 2
            weight 2
        }

        vrrp_instance VI_1 {
            interface ens18
            state BACKUP
            virtual_router_id 51
            priority 100
            advert_int 1
            authentication {
                auth_type PASS
                auth_pass Qwerty@1234
            }
            virtual_ipaddress {
                192.168.80.110
            }
            unicast_src_ip 192.168.80.102
            unicast_peer {
              192.168.80.100
              192.168.80.101
            }
            track_script {
                chk_haproxy
            }
        }
        ```

3. **Verify and start Keepalived**:
    ```sh
    keepalived -t
    systemctl start keepalived
    ip addr show dev ens18
    ```

### Step 3: Prepare All Nodes

1. **Disable swap**:
    ```sh
    swapoff -a
    sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
    ```

2. **Load necessary kernel modules**:
    ```sh
    tee /etc/modules-load.d/containerd.conf <<EOF
    overlay
    br_netfilter
    EOF

    modprobe overlay
    modprobe br_netfilter
    ```

3. **Set up required sysctl parameters**:
    ```sh
    tee /etc/sysctl.d/kubernetes.conf <<EOF
    net.bridge.bridge-nf-call-ip6tables = 1
    net.bridge.bridge-nf-call-iptables = 1
    net.ipv4.ip_forward = 1
    EOF

    sysctl --system
    ```

4. **Install necessary packages**:
    ```sh
    apt update
    apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates
    ```

5. **Configure and start containerd**:
    ```sh
    containerd config default | tee /etc/containerd/config.toml >/dev/null 2>&1
    sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
    systemctl restart containerd
    systemctl enable containerd
    ```

6. **Install Kubernetes components**:
    ```sh
    apt-get install -y apt-transport-https ca-certificates curl gpg
    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
    echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
    apt update
    apt install -y kubelet kubeadm kubectl
    apt-mark hold kubelet kubeadm kubectl
    kubeadm config images pull
    ```

### Step 4: Initialize the Kubernetes Cluster

1. **On `master01`, create the kubeadm configuration file**:
    ```sh
    vi kubeadm-config.yaml
    ```

    **Add the following content**:
    ```plaintext
    apiVersion: kubeadm.k8s.io/v1beta3
    kind: ClusterConfiguration
    controlPlaneEndpoint: "192.168.80.110:6443"
    ```

2. **Initialize the cluster**:
    ```sh
    kubeadm init --config=kubeadm-config.yaml --upload-certs
    ```

3. **Execute the commands from the output on the other control plane and worker nodes**.

### Step 5: Configure kubectl for the User

1. **Run these commands on the desired user**:
    ```sh
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    ```

### Step 6: Deploy the Network Plugin

1. **Deploy Calico on all nodes**:
    ```sh
    kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml
    ```
