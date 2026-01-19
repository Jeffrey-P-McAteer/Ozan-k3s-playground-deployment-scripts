
# k3s Playground for Ozan

OS: Ubuntu 25.04

Wildcard DNS: `*.ozan.jmcateer.com` and `ozan.jmcateer.com` will route to the machine's public ipv4 and ipv6 address.

Username: `ubuntu`

Access

```bash
# Least amount of config
ssh -i /path/to/my/id_ed25519 ubuntu@ozan.jmcateer.com

# If you're lazy like me, place the following in $HOME/.ssh/config (works on windows as well)
host ozan.jmcateer.com
  User ubuntu
  IdentityFile /path/to/my/id_ed25519

# Then you can do
ssh ozan.jmcateer.com
```

# Initial Setup

Use `ssh-copy-id` to copy a key you own to the server - this is the first & last time we use the VPS-provided password.

```bash
ssh-copy-id -i /path/to/my/id_ed25519 ubuntu@ozan.jmcateer.com
```

Remove password auth from the server.

```bash
sudo vim /etc/ssh/sshd_config
# Set "PasswordAuthentication no"
sudo systemctl restart sshd
```

# Installing & Using k3s

```bash
# Install k8s os dependencies
sudo apt update
sudo apt install -y curl ca-certificates gnupg lsb-release

# Misc other utilities
sudo apt-get install -y apache2-utils

# Open firewall for k3s resources we create
sudo ufw allow 80   # http
sudo ufw allow 443  # https
sudo ufw allow 6443 # kubectl

# Bring a Google contractor in to do the k3s install
curl -sfL https://get.k3s.io | sh -

# We are also going to add Helm for later use - this is a Kubernetes templating engine, because the tower of abstractions isn't tall enough until you re-create C macros.
sudo apt-get install -y curl gpg apt-transport-https
curl -fsSL https://packages.buildkite.com/helm-linux/helm-debian/gpgkey | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/helm.gpg] https://packages.buildkite.com/helm-linux/helm-debian/any/ any main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install -y helm

# Check that "control-plane" node (that's this machine! \o/) and "traefik-*" pod exists (internal kubernetes magic)
sudo kubectl get nodes
sudo kubectl get pods -A

# We now have a new file, which we will be re-using as creating multiple users/identities
# goes beyond "hello world" kubernetes.
# SECURITY: k3s.yaml CONTAINS PRIVATE KEY MATERIAL - it is a secret and must be treated as such!
# Copy /etc/rancher/k3s/k3s.yaml to your own $HOME/.kube/config and then you can use `kubectl` to directly manage resources
# You will need to change the line with "server: https://127.0.0.1:6443" to be "server: https://ozan.jmcateer.com:6443"

# Disable anonymous access & a few other decent security options. We're still exposing kubectl's API, but only using client SSL certificate auth
sudo vim /etc/rancher/k3s/config.yaml
# Add the following
cat <<EOF
kube-apiserver-arg:
  - anonymous-auth=false
  - service-account-lookup=false
  - token-auth-file=
EOF
# Restart k3s
sudo systemctl restart k3s

# Install cert-manager - this makes k3s automatically deploy SSL certs
sudo kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml

# Write down the definition of our ClusterIssuer - this is how kubernetes knows
# what you want it to do. In this instance, we're saying "create a resource named 'letsencrypt-http01' which other pods can ask to issue SSL certs"
vim letsencrypt-http01.yaml
cat <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-http01
spec:
  acme:
    email: ozan-k3s-playground@jmcateer.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-http01-account-key
    solvers:
    - http01:
        ingress:
          class: traefik
EOF
# Now we tell Kubernetes to create the resource
sudo kubectl apply -f letsencrypt-http01.yaml
# You can see the results with
sudo kubectl get clusterissuer


# We can test this setup by creating a "Hello World" Deployment and Service.
# kind: Ingress    => "I map External HTTP/S routes to internal Services"
# kind: Service    => "I map a Set of containers to a single routable resource"
# kind: Deployment =>  "I want to run N containers which collectively provide a service"
# The "---" lines are a shorthand for "treat this as-if it was a new .yaml file"

vim hello-world-service.yaml
cat <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-world
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-http01
spec:
  tls:
  - hosts:
    - hello-world.ozan.jmcateer.com
    secretName: hello-world-tls
  rules:
  - host: hello-world.ozan.jmcateer.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: hello-world
            port:
              number: 80

---

apiVersion: v1
kind: Service
metadata:
  name: hello-world
spec:
  selector:
    app: hello-world
  ports:
    - port: 80
      targetPort: 5678

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
      - name: hello
        image: hashicorp/http-echo
        args:
          - "-text=hello from ozan's k3s"
        ports:
          - containerPort: 5678

EOF

sudo kubectl apply -f hello-world-service.yaml

# Now open https://hello-world.ozan.jmcateer.com in a browser, curl, etc.
# It has an SSL cert and returns the result of your container, "image: hashicorp/http-echo" with the argument "-text=hello from ozan's k3s"
# If we had instead made that deployment a "image: syncfusion/word-processor-server", we'd be seeing their software running with no special configuration.


# One final adventure - we will be setting up a web UI to manage the cluster.
# This entails running a google web UI and creating HTTP basic auth over SSL,
# and we're just going to hard-code a username/password at this time to keep auth simple.
sudo kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
# See the created Pods
sudo kubectl get pods -n kubernetes-dashboard


# Import some Middleware resource definitions
kubectl apply -f https://raw.githubusercontent.com/traefik/traefik/v2.10/docs/content/reference/dynamic-configuration/kubernetes-crd-definition-v1.yml

# We will write down the service account, Ingress controller, and a middleware which checks the password
# as its own yaml file.
# The dashboard will end up being accessible at https://dashboard.ozan.jmcateer.com

vim dashboard-access-data.yaml
cat <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-http01
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
    traefik.ingress.kubernetes.io/router.middlewares: kubernetes-dashboard-auth@kubernetescrd
spec:
  tls:
  - hosts:
    - dashboard.ozan.jmcateer.com
    secretName: dashboard-tls
  rules:
  - host: dashboard.ozan.jmcateer.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: kubernetes-dashboard
            port:
              number: 443

---

apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: auth
  namespace: kubernetes-dashboard
spec:
  basicAuth:
    secret: dashboard-basic-auth

---

apiVersion: v1
kind: ServiceAccount
metadata:
  name: dashboard-user
  namespace: kubernetes-dashboard

---

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: dashboard-user-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: dashboard-user
  namespace: kubernetes-dashboard

EOF

# We can Create the referenced Kubernetes secret by running the following
sudo kubectl create secret generic dashboard-basic-auth --from-literal=users="$(htpasswd -nb ozan 'USERS_PASSWORD_HERE')" -n kubernetes-dashboard
# Then create the other resources
sudo kubectl apply -f dashboard-access-data.yaml




```



