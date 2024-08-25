# Kubernetes Install with Ansible

This repo was created from a [KubeExp](https://blog.mei-home.net/posts/kubernetes-cluster-setup/) blog, and it contains an ansible playbook to install a highly available kubernetes cluster on a set of nodes. This playbook is written for Debian 12 (Bookworm) with a non-root account and ssh, either VMs or bare metal, and will perform all of the steps needed to install kubernetes 1.30 with cri-o. I wasn't able to get kube-vip to work, so instead this sets up haproxy and keepalived as static pods to provide a virtual IP for the control plane, as demonstrated on the [kubeadm github](https://github.com/kubernetes/kubeadm/blob/main/docs/ha-considerations.md#options-for-software-load-balancing).

## Requirements

- Two or more Debian 12 (Bookworm) hosts
- ssh
- ansible
- python3

## Files to edit

1. Edit playbook `playbooks/k8s/0_k8s_sudo.yaml` with the username on the debian server.

2. Edit the IPs in `inventory/hosts.yaml` to match your environment. You can use IPs or hostnames in this file. This playbook assumes you will have 3 control nodes, and any number of worker nodes. You can just have one control node, although I haven't tested that configuration. It also supports one or more nodes with an nvidia or intel iGPU, and a coral TPU device.

3. Edit lines 59 and later in `templates/haproxy.cfg` for the correct IPs or hostnames of your control nodes.

4. If you have a node with an nvidia GPU, put it under the "k8s_gpu" section in the inventory file, and uncomment the `gpu_setup.yaml` line in the `run.yaml` playbook. If this node also has an intel iGPU, uncomment the `intel_setup.yaml` line in the `run.yaml` playbook.
Likewise, if you have a Coral PCIe TPU in that node, uncomment the `tpu_setup.yaml` line in the `run.yaml` playbook.

5. Edit line 17 and 26 of the `templates/keepalived.conf` file. Line 17 is the interface the control servers are listening on, and line 26 will be the virtual IP for the control plane. If you have DNS set up, you can put a hostname in front of the IP and use it in the following step.

6. Edit the `templates/kube-init-config.yaml` file, here you should set the controlPlaneEndpoint to the IP or hostname of the virtual IP you used for the control plane. You can also set clusterName here. You typically don't need to change the podSubnet or serviceSubnet, but if you do change the podSubnet, make sure you update the cilium command to match.

7. Edit the other commands in this README that use `k8sapi.example.com` with the virtual IP or hostname you used for the control plane.

## run playbook

```bash
ansible-playbook playbooks/k8s/run.yaml --ask-become-pass -v
```

## init the cluster on the first control node

```bash
ansible --become --ask-become-pass k8s_control_a -a "kubeadm init --upload-certs --config /etc/kubernetes/kube-init-config.yaml"
ansible --become --ask-become-pass k8s_control_a -a "kubeadm init phase upload-certs --upload-certs"
# Update these variables with the output from the above commands, they will be used later.
export TOKEN=example.token
export DISCOVERY_TOKEN_HASH=sha256:38bf13ef9985026a3fb71fea9ae95826cf8d84b02f300d481ba90a61f35504a6
export CERTIFICATE_KEY=13550350a8681c84c861aac2e5b440161c2b33a3e4f302ac680ca5b686de48de
```

## import kubeconfig

```bash
ansible --become --ask-become-pass k8s_control_a -a "cat /etc/kubernetes/admin.conf" > ~/.kube/admin-config
sed -i '/CHANGED/d' ~/.kube/admin-config
chmod 600 ~/.kube/admin-config
export KUBECONFIG=~/.kube/admin-config
# Verify this file doesn't have any strange characters at the beginning or the end
# TODO: make this command clean up the file automatically
```

## install cilium

```bash
cilium-cli install --set ipam.mode=cluster-pool --set ipam.operator.clusterPoolIPv4PodCIDRList=10.10.0.0/16 --set kubeProxyReplacement=true --set ingressController.enabled=true --set ingressController.loadbalancerMode=shared --version 1.16.1 --set encryption.enabled=true --set encryption.type=wireguard
```

## join other control nodes

```bash
ansible --become --ask-become-pass k8s_control_b -a "kubeadm join k8sapi.example.com:6443 --control-plane --certificate-key $CERTIFICATE_KEY --token $TOKEN --discovery-token-ca-cert-hash $DISCOVERY_TOKEN_HASH"
ansible --become --ask-become-pass k8s_control_c -a "kubeadm join k8sapi.example.com:6443 --control-plane --certificate-key $CERTIFICATE_KEY --token $TOKEN --discovery-token-ca-cert-hash $DISCOVERY_TOKEN_HASH"
```

## join worker nodes

```bash
ansible --become --ask-become-pass k8s_worker -a "kubeadm join k8sapi.example.com:6443 --token $TOKEN --discovery-token-ca-cert-hash $DISCOVERY_TOKEN_HASH"
```

## generate kubeconfig, role binding and namespace for user bob

```bash
ansible --become --ask-become-pass k8s_control_a -a "kubeadm kubeconfig user --client-name=bob" > ~/.kube/my-config
sed -i '/CHANGED/d' ~/.kube/my-config
chmod 600 ~/.kube/my-config
# Verify this file doesn't have any strange characters at the beginning or the end. It should probably end with an =. Mine had `I0824` followed by the time at the end that I had to delete manually.
# TODO: make this command clean up the file automatically

# Repeat this command for each namespace that you would like to give bob access to
ansible --become --ask-become-pass k8s_control_a -a "kubectl --kubeconfig /etc/kubernetes/admin.conf create namespace bob"
ansible --become --ask-become-pass k8s_control_a -a "kubectl --kubeconfig /etc/kubernetes/admin.conf create rolebinding bob-binding --clusterrole=admin --user=bob --namespace=bob"
```

## Flux setup

I have created another repo at [flux-k8s](https://github.com/jab416171/flux-k8s) with instructions on how to set up fluxcd on your cluster. This will allow you to manage your cluster with gitops. I have installed the kubernetes drivers necessary for nvidia GPU, intel iGPU, and coral TPU devices, as well as metallb, the prometheus monitoring stack, an nfs provisioner, and cert-manager. Once you have created a cluster with ansible, you should set up flux to install all of the other components you need to take advantage of the hardware in your cluster.
