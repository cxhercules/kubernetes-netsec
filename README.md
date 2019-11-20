# kubernetes-netsec
Kubernetes Presentation on CNI and Network policies

You can create allow rules, you are working with dafault deny otherwise.
Rules are additive and they are not joined by and but joined by or. 
Rules are namespaced 
You can use a namespace selector
Not a sliver bullet, more of defense in depth

## Network security gke cluster

In cloud shell run following:
```
echo '
## GCP K8S Deep Dive
export PROJECT_ID=$(gcloud config get-value project)
export COMPUTE_REGION=us-west1
gcloud config set project $PROJECT_ID
gcloud config set compute/region $COMPUTE_REGION
export COMPUTE_ZONE=us-west1-c
gcloud config set compute/zone $COMPUTE_ZONE
' >> ~/.bashrc

echo "source <(kubectl completion bash)" >> ~/.bashrc
bash -l    

gcloud services list --enabled
```

```
gcloud services enable \
  container.googleapis.com \
  compute.googleapis.com \
  containerregistry.googleapis.com \
  cloudbuild.googleapis.com \
  sourcerepo.googleapis.com \
  monitoring.googleapis.com \
  logging.googleapis.com \
  stackdriver.googleapis.com  
```

## Create gke cluster
```
gcloud container clusters create netsec --enable-network-policy
```
## Check status
```
gcloud container clusters describe netsec
```

## Gather credentials
```
gcloud container clusters get-credentials netsec
```

## Write policy but don't apply yet
```
cat > deny-egress.yaml <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: foo-deny-egress
spec:
  podSelector:
    matchLabels:
      app: foo
  policyTypes:
  - Egress
  egress:
  # allow DNS resolution
  - ports:
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP
EOF
```

If you don't see a command prompt, try pressing enter.
```
kubectl run --rm --restart=Never --image=alpine -i -t -l app=foo test -- ash
```

Run following:
```
/ # wget --timeout 1 -O- http://www.example.com
Connecting to www.example.com (93.184.216.34:80)
<!doctype html>
<html>
<head>
    <title>Example Domain</title>

    <meta charset="utf-8" />
    <meta http-equiv="Content-type" content="text/html; charset=utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <style type="text/css">
    body {
        background-color: #f0f0f2;
        margin: 0;
        padding: 0;
        font-family: -apple-system, system-ui, BlinkMacSystemFont, "Segoe UI", "Open Sans", "Helvetica Neue", Helvetica, Arial, sans-serif;

    }
    div {
        width: 600px;
        margin: 5em auto;
        padding: 2em;
        background-color: #fdfdff;
        border-radius: 0.5em;
        box-shadow: 2px 3px 7px 2px rgba(0,0,0,0.02);
    }
    a:link, a:visited {
        color: #38488f;
        text-decoration: none;
    }
    @media (max-width: 700px) {
        div {
            margin: 0 auto;
            width: auto;
        }
    }
    </style>
</head>

<body>
<div>
    <h1>Example Domain</h1>
    <p>This domain is for use in illustrative examples in documents. You may use this
    domain in literature without prior coordination or asking for permission.</p>
    <p><a href="https://www.iana.org/domains/example">More information...</a></p>
</div>
</body>
</html>
-                    100% |****************************************************************************************************************************************************************************************************|  1256  0:00:00 ETA
/ #
```

## Apply policy
```
kubectl apply -f deny-egress.yaml
```

## After applying can't access site but dns still works
```
$ kubectl run --rm --restart=Never --image=alpine -i -t -l app=foo test -- ash
If you don't see a command prompt, try pressing enter.
/ # wget --timeout 1 -O- http://www.example.com
Connecting to www.example.com (93.184.216.34:80)
wget: download timed out
```

## Try another container without label
```
$kubectl run --rm --restart=Never --image=alpine -i -t  test -- ash
wget --timeout 1 -O- http://www.example.com
```

## Denying all traffic by matching to it
```
cat > default-deny-all.yaml <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: 
    name: default-deny-all
spec:
    # select all
    podSelector: {}
    # Nothing is whitelisted
    ingres: []
EOF
```

## Frontend can't talk to db, but backend can
```
cat > allow-app-flow.yaml <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: 
    name: allow-app-flow
spec:
    podSelector:
        matchLabels:
            app: foo
            tier: db
    ingres:
    - from:
      - podSelector:
        matchLabels:
            app: foo
            tier: backend
      ports:
      - port: 3306
        protocol: TCP
EOF
```

## You can also allow traffic using cidr blocks
```
from:
- ipBlock:
    cidr: 10.56.0.0/16

```
or 
```
from:
- ipBlock:
    cidr: 10.0.0.0/8
    except:
    - 10.11.12.0/24
```

## Deny All egress traffic
```
cat > default-deny-all-egress.yaml <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: 
    name: default-deny-all-egress
spec:
    # select all
    podSelector: {}
    policyTypes:
    - Egress
    # Nothing is whitelisted
    egress: []
EOF
```

## Only allow fronted to go to backend
If we did not have allow dns above we would need to add it now. 

```
cat > front-to-back-egress.yaml <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: 
    name: front-to-back-egress
spec:
    podSelector:
        matchLabels:
            tier: fronted
    policyTypes:
    - Egress
    egress:
    - to:
      - podSelector:
        matchLabels:
            tier: backend
EOF
```

## Allow only egress to internal traffic and not external world
```
cat > allow-local-egress.yaml <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: 
    name: allow-local-egress
spec:
    # select all
    podSelector: {}
    policyTypes:
    - Egress
    # all namespaces
    egress:
    - to:
      - namespaceSelector: {}
EOF
```

# Best practices
- Apply default deny for ingress and egress and start allowing what is needed
- Test your assumptions, test your rules


# Reference:
- https://github.com/GoogleCloudPlatform/gke-network-policy-demo
- https://www.youtube.com/watch?v=3gGpMmYeEO8&t=0m0s
- https://www.altoros.com/blog/kubernetes-networking-writing-your-own-simple-cni-plug-in-with-bash/
- https://github.com/ahmetb/kubernetes-network-policy-recipes
- https://medium.com/better-programming/kubernetes-a-detailed-example-of-deployment-of-a-stateful-application-de3de33c8632
- https://itnext.io/kubernetes-network-deep-dive-7492341e0ab5
