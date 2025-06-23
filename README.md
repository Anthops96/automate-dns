# automate-dns
Automate creation of Secrets: 
Kubernetes manifests for the automatic creation of Secrets, implemented on the generated domains of the different components that are part of Elasticsearch, through a Cronjob, using Service Account and RBAC.

## Create Service Account Resource
This resource is useful to authenticate with the Kubernetes API, necessary to execute actions such as CREATE, GET, etc., which will be declared later in the Role resource.
```sh
nano serviceaccount.yaml
kubectl apply -f serviceaccount.yaml
```

## Create the Role Resource
It allows us to declare the permissions that the service account will be able to perform in the assigned namespace, in this case on the secret resource.
```sh
nano  role.yaml
kubectl apply -f role.yaml
```

## Assign Role to Service Account using Role Binding resource
Links the service account to the role, granting permissions in the respective namespace
```sh
nano rb.yaml
kubectl apply -f rb.yaml
```
## Deploy Cronjob
This resource creates a job that we will execute manually, every time that we add a new domain, because of this we will not schedule it. It is made up of two containers, which are executed in order, the first is generated with the ElasticSearch image, fulfilling the function of creating the pk12 files that contain the certificates and private keys, generated from the added domains.
```yaml
          containers:
          - name: certificate-generator  # Initial container of the job
            image: elasticsearch/elasticsearch:8.17.0
            securityContext:
              runAsUser: 0
            command:      # Generate pk12 files from domain list
            - /bin/bash
            - -c
            - |
              set -e
              declare -A fqdns   
              fqdns=(
                  [1]='es-ingest'
                  [2]='es-coordinator'
                  [3]='fleet-server')
              rm -rf /certs/elastic-certificates.zip
              echo -e 'instances:' > /certs/instances.yml
              for fqdn in "${fqdns[@]}"; do rm -rf /certs/$fqdn/; done
              for fqdn in "${fqdns[@]}"; do echo -e "  - name: \"$fqdn\"\n    dns:\n      - \"$fqdn.example.com\"" >> /certs/instances.yml; done
              echo "Generating node certificates..."
              bin/elasticsearch-certutil cert --days 3600 --multiple --in /certs/instances.yml --ca-cert /certs/ca/ca.crt --ca-key /certs/ca/ca.key --silent --ca-pass "Password" --pass "Password" --out /certs/elastic-certificates.zip
              unzip /certs/elastic-certificates.zip -d /certs
              ls -la /certs
              touch /certs/step1_done
            volumeMounts:
            - name: certs
              mountPath: /certs
```

 The second container generated from the Ubuntu Noble image creates the certificates and private keys through the pk12 files, which it uses to create the Secrets.


```yaml
          - name: secret-creator  # Containers that end the job
            image: ubuntu:noble
            securityContext:
              runAsUser: 0
            command:  # Generate certificates from pk12 and creation of secrets
            - /bin/bash
            - -c
            - |
              set -e
              echo "Types: deb" > /etc/apt/sources.list.d/ubuntu.sources
              echo "URIs: http://archive.ubuntu.com/ubuntu/" >> /etc/apt/sources.list.d/ubuntu.sources
              echo "Suites: noble noble-updates noble-backports" >> /etc/apt/sources.list.d/ubuntu.sources
              echo "Components: main restricted universe multiverse" >> /etc/apt/sources.list.d/ubuntu.sources
              echo "Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg" >> /etc/apt/sources.list.d/ubuntu.sources

              echo "Types: deb" >> /etc/apt/sources.list.d/ubuntu.sources
              echo "URIs: http://archive.ubuntu.com/ubuntu/" >> /etc/apt/sources.list.d/ubuntu.sources
              echo "Suites: noble noble-updates noble-backports" >> /etc/apt/sources.list.d/ubuntu.sources
              echo "Components: main restricted universe multiverse" >> /etc/apt/sources.list.d/ubuntu.sources
              echo "Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg" >> /etc/apt/sources.list.d/ubuntu.sources
              apt update
              apt install openssl -y
              apt update && apt install -y openssl curl
              while [ ! -f /certs/step1_done ]; do sleep 1; done;
              rm -rf /certs/step1_done
              declare -A fqdns
              fqdns=(
                  [1]='es-ingest'
                  [2]='es-coordinator'
                  [3]='fleet-server')
              echo "Generating cert and key from .p12..."
              for fqdn in "${fqdns[@]}"; do openssl pkcs12 -in /certs/$fqdn/$fqdn.p12 -out /certs/$fqdn/$fqdn.key -nocerts -nodes -password pass:Password -passout pass:Password; done
              for fqdn in "${fqdns[@]}"; do openssl pkcs12 -in /certs/$fqdn/$fqdn.p12 -out /certs/$fqdn/$fqdn.crt -clcerts -nokeys -password pass:Password -passout pass:Password; done
              for fqdn in "${fqdns[@]}"; do
                echo "[+] Creating secret for $base"
                CRT=$(base64 -w 0 "/certs/$fqdn/$fqdn.crt")
                KEY=$(base64 -w 0 "/certs/$fqdn/$fqdn.key")
                echo '{
                  "apiVersion": "v1",
                  "kind": "Secret",
                  "metadata": {
                    "name": "tls-'"$fqdn"'"
                  },
                }' > /tmp/secret.json

                curl -k -X POST \
                  --header "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
                  --header "Content-Type: application/json" \
                  --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
                  --data-binary @/tmp/secret.json \
                  "https://${KUBERNETES_SERVICE_HOST}/api/v1/namespaces/${POD_NAMESPACE}/secrets" || \
                {
                  echo "Updating existing secret..."
                  curl -k -X PUT \
                    --header "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
                    --header "Content-Type: application/json" \
                    --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
                    --data-binary @/tmp/secret.json \
                    "https://${KUBERNETES_SERVICE_HOST}/api/v1/namespaces/${POD_NAMESPACE}/secrets/tls-$fqdn"
                }
              done
              echo "[✓] All secrets created/updated"
```

```sh
nano final_cronjob.yaml
kubectl apply -f final_cronjob.yaml
#Check status of the pod
kubectl get pods --selector=job-name=elastic-certificates-secrets-generator
```
