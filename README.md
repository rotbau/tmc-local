# TMC Self-Managed Install


## DNS Records

Create the following A records in your DNS

```
<my-tmc-dns-zone>
alertmanager.<my-tmc-dns-zone>
auth.<my-tmc-dns-zone>
blob.<my-tmc-dns-zone>
console.s3.<my-tmc-dns-zone>
gts-rest.<my-tmc-dns-zone>
gts.<my-tmc-dns-zone>
landing.<my-tmc-dns-zone>
pinniped-supervisor.<my-tmc-dns-zone>
prometheus.<my-tmc-dns-zone>
s3.<my-tmc-dns-zone>
tmc-local.s3.<my-tmc-dns-zone>
```
## Download Tanzu Mission Control Self Managed Bundle

1. Download Tanzu Mission Control (Self-Managed) from [support.broadcom.com](https://support.broadcom.com).  Current Bundle is 1.2.0
2. Download tanzu CLI from [Tanzu CLI Github](https://github.com/vmware-tanzu/tanzu-cli/blob/main/docs/quickstart/install.md).  Can use pre-build release.  Only 1.1.0 CLI technically supported with TMC SM

## Install Tanzu CLI

Follow instructions on the tanzu cli documentation or tanzu cli github to install the CLI and plugins.  You will need the tanzu package plugings

## Create Registry Project and Validate

1. Create a project in your registry (ex. tmcsm).  Project needs to be public as for version 1.0.1
2. If you have a self signed certificate for your registry make sure the ca has been added to /etc/ssl/certs/{registry.crt} of your linux jumpbox.  Note certificate needs to have SAN field I believe based on my testing
3. Validate you have login credentials and test login to registy `docker login registry.fqdn.com`

## Push Images to Registry
```
mkdir tmcsm
tar xvf bundle-1.2.0.tar -c tmcsm/
cd tmcsm
export user="{harbor-user}"
export password="{harbor-password}"
./tmc-sm push-images harbor --project harbor.vdoubleb.com/tmcsm --username $user --password $password
```
## Create TKGs Workload Cluster

### Create VM Class

1. Create VM Class called tmc-vm with 4 vCPU and 8 GB RAM
2. Create vSphere Namespace tmc and make sure tmc-vm VM Class is added to namespace

### Create Cluster Trusted CA Secret (Required if using self signed CA)

1. Get the ca.crt for the CA that signed your registry.  If you don't have it handy you can use this command to retrieve it.  May be differnt for non-harbor registries
```
wget -O harbor-ca.crt https://harbor.vdoubleb.com/api/v2.0/systeminfo/getcert --no-check-certificate
```
2. Create cluster trusted secret manifest template.
```
cat > svc-tmcsm-user-trusted-ca-secret.yaml <<-EOF
apiVersion: v1
kind: Secret
metadata:
  name: svc-tmcsm-user-trusted-ca-secret
  namespace: tmc
data:
  harbor-ca: TFMwdExTMUNSVWRKVGlCRFJWSlVTVVpKUTBGVVJTMHRMUzB0Q2sxSlNVWnhSRU5EUVRWRFowRjNTVUpCWjBsQ1FVUkJUa0puYTNGb2EybEhPWGN3UWtGUk1FWkJSRUpzVFZGemQwTlJXVVJXVVZGSFJYZEtWbFY2UlZNS1RVSkJSMEV4VlVWRFFYZEtWRmRzZFdKdFZu
  ............
type: Opaque
EOF
```
3. Edit svc-tmcsm-user-trusted-ca-secret.yaml and replace value under harbor-ca with the double base64 encoded ca.crt  
```
cat harbor-ca.crt | base64 -w 0 > harbor-b64.txt
cat harbor-b64.txt | base64 -w 0 > harbor-doubleb64.txt
Take value from harbor-doubleb64.txt and enter into svc-tmcsm-user-trusted-ca-secret.yaml under harbor-ca:
```
5. Create svc-tmcsm-user-trusted-ca-secret in vsphere namesapce where cluster will be created (ex tmc)
```
kubectl apply -f svc-tmcsm-user-trusted-ca-secret.yaml
```
### Create kappcontrollerconfig manifest (Required if using self signed CA)

Creating a kappcontrollerconfig prior to TKG workload cluster creation will assume kapp will be able to sync packages / repositories from your registry

1. Create kappcontollerconfig manifest file
```
export clustername="svc-tmcsm"

cat > $clustername-kapp-controller-config.yaml <<-EOF
apiVersion: run.tanzu.vmware.com/v1alpha3
kind: KappControllerConfig
metadata:
  name: $clustername-kapp-controller-package
  namespace: tmc
spec:
  kappController:
    config:
      caCerts: |
        -----BEGIN CERTIFICATE-----
        MIIFqDCCA5CgAwIBAgIBADANBgkqhkiG9w0BAQ0FADBlMQswCQYDVQQGEwJVUzES
        MBAGA1UECAwJTWlubmVzb3RhMRQwEgYDVQQHDAtNaW5uZWFwb2xpczEVMBMGA1UE
        ...........
        -----END CERTIFICATE-----
    createNamespace: false
    deployment:
      apiPort: 10100
      concurrency: 4
      hostNetwork: true
      metricsBindAddress: "0"
      priorityClassName: system-cluster-critical
      tolerations:
      - key: CriticalAddonsOnly
        operator: Exists
      - effect: NoSchedule
        key: node-role.kubernetes.io/control-plane
      - effect: NoSchedule
        key: node-role.kubernetes.io/master
      - effect: NoSchedule
        key: node.kubernetes.io/not-ready
      - effect: NoSchedule
        key: node.cloudprovider.kubernetes.io/uninitialized
        value: "true"
    globalNamespace: tkg-system
  namespace: tkg-system
EOF
```
2. Edit svc-tmcsm-kapp-controller-config.yaml and replace value under config.caCerts with your harbor CA value
3. Optionally if you need to set proxy or other configs for kapp use the following command to see options
```
kubectl explain kappcontrollerconfig.spec.kappController.config
```
4. Change context to vsphere namespace where cluster will be created (ex tmc)
5. Create kappcontrollerconfig
```
kubectl apply -f svc-tmcsm-kapp-controller-config.yaml
kubectl get kappcontrollerconfig

NAME                                NAMESPACE    GLOBALNAMESPACE   SECRETNAME
svc-tmcsm-kapp-controller-package   tkg-system   tkg-system
```
### Create TKGs Workload Cluster Manifest

```
cat > svc-tmcsm-cluster.yaml <<-EOF
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: svc-tmcsm
  namespace: tmc
spec:
  clusterNetwork:
    services:
      cidrBlocks: ["198.51.100.0/12"]
    pods:
      cidrBlocks: ["192.168.0.0/16"]
    serviceDomain: "cluster.local"
  topology:
    class: tanzukubernetescluster
    version: v1.26.13---vmware.1-fips.1-tkg.3
    controlPlane:
      replicas: 1
      metadata:
        annotations:
          run.tanzu.vmware.com/resolve-os-image: os-name=ubuntu
    workers:
      machineDeployments:
        - class: node-pool
          name: node-pool-1
          replicas: 3
          metadata:
            annotations:
              run.tanzu.vmware.com/resolve-os-image: os-name=ubuntu
    variables:
      - name: vmClass
        value: tmc-vm
      - name: storageClass
        value: vsan-default-storage-policy
      - name: defaultStorageClass
        value: vsan-default-storage-policy
      - name: trust
        value:
          additionalTrustedCAs:
          - name: harbor-ca
      - name: nodePoolVolumes
        value:
        - name: containerd
          capacity:
            storage: 20Gi
          mountPath: /var/lib/containerd
          storageClass: vsan-default-storage-policy
        - name: kubelet
          capacity:
            storage: 20Gi
          mountPath: /var/lib/kubelet
          storageClass: vsan-default-storage-policy
      - name: controlPlaneVolumes
        value:
        - name: containerd
          capacity:
            storage: 20Gi
          mountPath: /var/lib/containerd
          storageClass: vsan-default-storage-policy
        - name: kubelet
          capacity:
            storage: 20Gi
          mountPath: /var/lib/kubelet
          storageClass: vsan-default-storage-policy
      - name: clusterEncryptionConfigYaml
        value: |
          apiVersion: apiserver.config.k8s.io/v1
          kind: EncryptionConfiguration
          resources:
            - resources:
                - secrets
              providers:
                - aescbc:
                    keys:
                      - name: key1
                        secret: QiMg...............
                - identity: {}
EOF
```
1. Edit name, namespace, vmClass, storageClass, secret and trust with correct values for your environment

### Create Cluster
1. Run TKG Workload Cluster mainifest
```
kubectl apply -f svc-tmcsm-cluster.yaml
```

## Install Cert Manager and Cluster Issuer

### Install Cert Manager Package

1. Authenticate to svc-tmcsm Tanzu Cluster and make sure kubectl context is set to this cluster
2. Add VMware Reposistory for Packages
```
tanzu package repository add tanzu-standard --url projects.registry.vmware.com/tkg/packages/standard/repo:v2.2.0 --namespace tkg-system
tanzu package repository list -A
tanzu package available list -A
tanzu package available list cert-manager.tanzu.vmware.com -A
kubectl create ns tkg-packages
tanzu package install cert-manager --package cert-manager.tanzu.vmware.com --namespace tkg-system --version 1.10.2+vmware.1-tkg.1
tanzu package installed list -A
```
3. Make sure cert-manager is listed as Reconcile Succeeded

### Cluster Issuer or Secrets

Version 1.2.0 can use a local cluster issue or self-supplied certificates in the form of secrets

### Cluster Issuer
1. Create a self signed CA for TMC Cluster Issuer
```
export DOMAIN="*.tmc.vdoubleb.com"
export SUBJ="/C=US/ST=Minnesota/L=Minneapolis/O=VMware, Inc./OU=Tanzu/CN=${DOMAIN}"
openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -key ca.key -sha512 -days 3650 -out tmcsm-ca.crt -subj "$SUBJ"
```
2. Create Server Private Key and CSR
```
openssl genrsa -out server-app.key 4096
openssl req -sha512 -new -subj "$SUBJ" -key server-app.key -out server-app.csr
```
3. Create Extensions file for Certificate
```
cat > v3.ext <<-EOF
authorityKeyIdentifier=keyid,issuer
  basicConstraints=CA:FALSE
  keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
  extendedKeyUsage = serverAuth
  subjectAltName = @alt_names
  [alt_names]
  DNS.1=${DOMAIN}
EOF
```
4. Create Server Certificate
```
openssl x509 -req -sha512 -days 3650 -extfile v3.ext -CA tmcsm-ca.crt -CAkey ca.key -CAcreateserial -in server-app.csr -out server-app.crt
```
Inspect certificate with `openssl x509 -in server-app.crt -text -noout`

5. Copy Certificate to Jump Box
```
sudo cp tmcsm-ca.crt /etc/ssl/certs/
```
6. Create TLS Secret for Cert-Manager
```
kubectl create secret tls local-ca --key ca.key --cert tmcsm-ca.crt -n cert-manager
```
7. Create Cluster Issuer File
```
cat > local-issuer.yaml <<-EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: local-issuer
  namespace: cert-manager
spec:
  ca:
    secretName: local-ca
EOF
```
8. Apply Cluster Isser YAML
```
kubectl apply -f local-issuer.yaml
```
```
kubectl get clusterissuer -A

NAME           READY   AGE
local-issuer   True    8s
```

### Self Signed Certificates
1. Create certifcates for each of the 6 service listed in the documentation (landing service, minio-tls, pinniped, server, stack and tenancy).  Do not use the shared/wildcard option in the documentation.  There is a but with contour that will break TMC SM.

## Deploy TMC Self-Managed

### Add TMC SM Package Repository

1. Change context to svc-tmcsm Tanzu cluster
2. Create tmc-local namespace and set pod-security if on 1.26 or greater
```
kubectl create ns tmc-local
kubectl label ns tmc-local pod-security.kubernetes.io/enforce=privileged
```
3. Add TMC SM Package Repository to tmc-local namespace
```
tanzu package repository add tanzu-mission-control-packages --url "harbor.vdoubleb.com/tmcsm/package-repository:1.2.0" --namespace tmc-local
tanzu package repository list -n tmc-local
tanzu package repository get tanzu-mission-control-packages -n tmc-local
tanzu package available list  -n tmc-local
tanzu package available get tmc.tanzu.vmware.com -n tmc-local
```
### Create TMC Self-Managed Configuration File

Note as of TMC Self Managed 1.2 support LDAP / Active Directory and OIDC options for Idenity Management. See the values-ad.yaml and values.keycload.yaml examples in the repository for more information on how to configure these options along with the offical documentation.

1. You can view all the possible Key Values on the [Offical Documentation](https://docs.vmware.com/en/VMware-Tanzu-Mission-Control/1.2/tanzumc-sm-install/install-config-keys.html)
2. You can also use the tanzu package available get command
```
tanzu package available get tmc.tanzu.vmware.com/1.2.0 -n tmc-local --values-schema
```
2. Create Vales.yaml Configuration File
```
cat > values.yaml <<-EOF
harborProject: "harbor.vdoubleb.com/tmcsm"
dnsZone: "tmc.vdoubleb.com"
supportFlags:
  - tmc-integration-environment
clusterIssuer: "local-issuer"
oidc:
  issuerType: "pinniped"
  issuerURL: "https://keycloak.vdoubleb.com:8443/realms/tmcsm"
  clientID: "tmcsm"
  clientSecret: "b6CONU7xBHJTEpCxo1nEw2mPb76FPRJ7"
trustedCAs:
  local-ca.pem: |
    -----BEGIN CERTIFICATE-----
    MIIFmjCCA4KgAwIBAgICEAAwDQYJKoZIhvcNAQENBQAwZTELMAkGA1UEBhMCVVMx
    EjAQBgNVBAgMCU1pbm5lc290YTEUMBIGA1UEBwwLTWlubmVhcG9saXMxFTATBgNV
    .......
    -----END CERTIFICATE-----
  idp-ca.pem: |
    -----BEGIN CERTIFICATE-----
    MIID6zCCAtOgAwIBAgIUO/AHkPOAB1rrov5kKHPdEuxtYccwDQYJKoZIhvcNAQEL
    BQAwcTELMAkGA1UEBhMCVVMxCzAJBgNVBAgMAk1OMRQwEgYDVQQHDAtNaW5uZWFw
    .......
    -----END CERTIFICATE-----
  harbor-ca.pem: |
    -----BEGIN CERTIFICATE-----
    MIIFqDCCA5CgAwIBAgIBADANBgkqhkiG9w0BAQ0FADBlMQswCQYDVQQGEwJVUzES
    MBAGA1UECAwJTWlubmVzb3RhMRQwEgYDVQQHDAtNaW5uZWFwb2xpczEVMBMGA1UE
    .......
    -----END CERTIFICATE-----
contourEnvoy:
  serviceType: LoadBalancer
pinnipedExtraEnvVars: []
#alertmanager: # needed only if you want to turn on alerting
#  criticalAlertReceiver:
#    slack_configs:
#    - send_resolved: false
#      api_url: https://hooks.slack.com/services/...
#      channel: '#<slack-channel-name>'
minio:
  username: root
  password: "VMware123!"
postgres:
  userPassword: "VMware123!"
  maxConnections: 300
telemetry:
  ceipOptIn: false
  eanNumber: ean-not-specified
  ceipAgreement: false
EOF
```
3. Replace local-ca.pem with the CA.crt that signed the local-issuer certificate
4. Replace the ida-ca.pem with the CA.crt that signed the IDP solution (WS1, Keycloak, Ping, Okta) if using a self-signed certificate
5. Replace the harbor-ca.pem with the CA.crt that signed the harbor registry
6. Configure OIDC parameters for provider
7. Configure harbor FQDN and project
8. Configure dnsZone

### Deploy TMC Self-Managed

1. Deploy TMC Self-Managed
```
tanzu package install tanzu-mission-control -p tmc.tanzu.vmware.com --version "1.2.0" --values-file values.yaml --namespace tmc-local
```
2. Optional: In another SSH session Watch tmc-local namespace.  Note it is expected some pods will be in configerror or crashloopbackoff at different stages of the install
```
kubectl get po -n tmc-local -w
```
3. Watch for deployment to complete
```
            | 3:38:52AM: ---- waiting complete [28/28 done] ----
            | Succeeded
10:38:52PM: Deploy succeeded
```
4. View reconsiliation status
```
tanzu package installed list -n tmc-local

  NAME                          PACKAGE-NAME                                       PACKAGE-VERSION  STATUS
  contour                       contour.bitnami.com                                15.5.4           Reconcile succeeded
  kafka                         kafka.bitnami.com                                  26.8.6           Reconcile succeeded
  kafka-topic-controller        kafka-topic-controller.tmc.tanzu.vmware.com        0.0.30           Reconcile succeeded
  minio                         minio.bitnami.com                                  13.5.2           Reconcile succeeded
  pinniped                      pinniped.bitnami.com                               1.10.2           Reconcile succeeded
  postgres                      tmc-local-postgres.tmc.tanzu.vmware.com            0.0.135          Reconcile succeeded
  postgres-endpoint-controller  postgres-endpoint-controller.tmc.tanzu.vmware.com  0.1.57           Reconcile succeeded
  redis                         redis.bitnami.com                                  18.16.2          Reconcile succeeded
  s3-access-operator            s3-access-operator.tmc.tanzu.vmware.com            0.1.31           Reconcile succeeded
  tanzu-mission-control         tmc.tanzu.vmware.com                               1.2.0            Reconcile succeeded
  tmc-local-monitoring          monitoring.tmc.tanzu.vmware.com                    0.0.18           Reconcile succeeded
  tmc-local-stack               tmc-local-stack.tmc.tanzu.vmware.com               0.0.24011        Reconcile succeeded
  tmc-local-stack-secrets       tmc-local-stack-secrets.tmc.tanzu.vmware.com       0.0.24011        Reconcile succeeded
  tmc-local-support             tmc-local-support.tmc.tanzu.vmware.com             0.0.24011        Reconcile succeeded

```
### Obtain Contour IP and Httpproxy Information

1. Obtain IP of Contour Envoy Service
```
kubectl get svc -n tmc-local |grep envoy

contour-envoy                                      LoadBalancer   198.56.79.190    10.0.103.15   80:30347/TCP,443:30576/TCP   13m
contour-envoy-metrics                              ClusterIP      None             <none>        8002/TCP                     13m
```
2. Make sure all of your DNS records are set to the external IP (10.0.103.15)
3. View httpproxy objects for TMC Self Managed
```
kubectl get httpproxy -n tmc-local

NAME                              FQDN                                   TLS SECRET   STATUS   STATUS DESCRIPTION
auth-manager-server               auth.tmc.vdoubleb.com                  server-tls   valid    Valid HTTPProxy
minio-api-proxy                   s3.tmc.vdoubleb.com                    minio-tls    valid    Valid HTTPProxy
minio-bucket-proxy                tmc-local.s3.tmc.vdoubleb.com          minio-tls    valid    Valid HTTPProxy
minio-console-proxy               console.s3.tmc.vdoubleb.com            minio-tls    valid    Valid HTTPProxy
pinniped-supervisor               pinniped-supervisor.tmc.vdoubleb.com                valid    Valid HTTPProxy
stack-http-proxy                  tmc.vdoubleb.com                       stack-tls    valid    Valid HTTPProxy
tenancy-service-http-proxy        gts.tmc.vdoubleb.com                                valid    Valid HTTPProxy
tenancy-service-http-proxy-rest   gts-rest.tmc.vdoubleb.com                           valid    Valid HTTPProxy
```
### Log in to TMC Self-Managed UI

1. From the httpproxy list you will notice the tmc.vdoubleb.com entry, this is the URL for the TMC SM UI
2. Can use curl to test
```
curl -ik https://tmc.vdoubleb.com
Should see a 200 OK
```
3. Open TMC Self-Managed in Browser

## Updating or Deleting TMC Self-Managed

### Update changes to values.yaml
```
tanzu package installed update tanzu-mission-control -p tmc.tanzu.vmware.com --version "1.2.0" --values-file values.yaml --namespace tmc-local
```
### Uninstall TMC Self-Managed
```
tanzu package installed delete tanzu-mission-control -n tmc-local
```
## Register TKGs Management Cluster in TMC Self-Managed

### Start Management Cluster Registration in TMC UI

1. Navigate to Administration -> Management Clusters -> Register Management Cluster -> vSphere with Tanzu
- Name: Suggest you use name of supervisor cluster
- Default Cluster Group: Select Default
- Click Next
2. Registration Link will be generated.  Use this link in the TMC Agent Install Manifest


### Create TMC Agent Config Manifest

1. Get the supervisor cluster Tanzu Mission Control Namespace.  From Supervisor context run
```
kubectl get ns |grep svc-tmc

svc-tmc-c8                                  Active   9d
```
2. Create agent config yaml that contains the CA your local issuer was signed with
```
cat > tmc-agent-config.yaml <<-EOF
apiVersion: "installers.tmc.cloud.vmware.com/v1alpha1"
kind: "AgentConfig"
metadata:
  name: "tmc-agent-config"
  namespace: "svc-tmc-c8"
spec:
  caCerts: |-
    -----BEGIN CERTIFICATE-----
    CA-CERT-CONTENTS
    -----END CERTIFICATE-----
  allowedHostNames:
    - "TMC-SM-HOSTNAME"
EOF
```
3. Replace the namespace with the value obtained in step 1, replace caCerts with your CA that signed local issuer, update allowedHostNames with the FQDN of your TMC Self-Managed instance
4. Apply agent config from supervisor cluster context
```
kubectl apply -f tmc-agent-config.yaml
```
### Create TMC Agent Install Manifest

1. Create agent install manifest
```
cat > tmc-registration.yaml <<-EOF
apiVersion: installers.tmc.cloud.vmware.com/v1alpha1
kind: AgentInstall
metadata:
  name: tmc-agent-installer-config
  namespace: TMC-NAMESPACE
spec:
  operation: INSTALL
  registrationLink: TMC-REGISTRATION-URL
EOF
```
2. Replace namespace with the name of the tmc namepace you used in the agent-config section.  Replace the TMC-REGISTRATION-URL with the URL you obtained for the TMC UI when you started Management Cluster Registration
3. Register Supervisor.  Run following from the supervisor cluster context
```
kubectl create -f tmc-registration.yaml
```
4. Verify Install.  Look for status changing from INSTALLATION_IN_PROGRESS to INSTALLED 
```
kubectl -n svc-tmc-c8 describe agentinstall tmc-agent-installer-config
```
5. Wait a few minutes and then return to TMC UI and click Verify Cluster or View Management Cluster.  It is normal for this process to take several minutes and cluster to be unknown state. Extensions and Component Health will take some time to go green.

# Install Tanzu Packages to TMC Local

1. Pull down TMC Packages from projects.registry.vmware.com using imgpkg
```
imgpkg copy --registry-ca-cert-path=harborca.pem \
-b projects.registry.vmware.com/tkg/packages/standard/repo:v2.2.0_update.2\
 --to-repo harbor.vdoubleb.com/tmcsm/498533941640.dkr.ecr.us-west-2.amazonaws.com/packages/standard/repo
 ```
