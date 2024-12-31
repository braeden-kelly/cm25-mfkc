# my-first-kubernetes
An interactive workshop for learning a few Kubernetes basics, for developers that already know the basics of Docker and containers.

- signup and gen envs
- bookinfo via docker
  - discuss k8s capabilities
    - service discovery
    - networking
    - failover/self-healing
    - scaling/load balancing
    - rollouts/rollbacks
    - resource management
- deploy bookinfo in k3s by hand
  - pods
  - kubectl create
  - kubectl expose
  - kubectl delete
  - deployments
- edit bookinfo
  - service
    - ClusterIP
    - NodePort
    - LoadBalancer
    - Ingress
- Nodes
- K3s agents
  - namespaces
  - kubectl apply
  - checkov

- Others
  - kubectl exec


Deployment rollout, CrashLoopBackOff
readiness-liveness
Volumes
Namespaces - Multi-tenant

Demo: AWS EKS, autoscaling, Cilium, Hubble, Network Policies

Template `v1` to `v2` in two places

For admins, do [Kubernetes the Hard
Way](https://github.com/kelseyhightower/kubernetes-the-hard-way). Then never do
it without a tool again.

For practice,

- [Tutorials](https://kubernetes.io/docs/tutorials/)
- [KodeKloud Kubernetes Free Labs](https://kodekloud.com/free-labs/kubernetes).

More learning,

- [LinuxFoundationX: Introduction to
  Kubernetes](https://www.edx.org/learn/kubernetes/the-linux-foundation-introduction-to-kubernetes)
- [TechWorld with Nana](https://youtu.be/s_o8dwzRlu4?si=T38bQK3OhwbIEzJ7