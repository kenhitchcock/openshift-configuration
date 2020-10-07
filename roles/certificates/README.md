# Ansible Role for Ingress Certificates

## Table of Contents
1. [OpenShift Ingress Traffic Diagram](#paragraph0)
2. [Ingress Certificate Role ](#paragraph1)

## OpenShift Ingress Traffic Diagram <a name="paragraph0"></a>

![](images/diagram.png)

All Worker VMs are available in Load Balancer backend pool even if only two of them have OpenShift Router Pods running (they can be recreated on another VM after restart). Traffic is sent to the first available pool member VM.

Ingress Controller creates OpenShift Router Pods which are Ingress points for traffic to enter the cluster. By default, Ingress Controller attach Certificate Operator precreated Self-Signed certificate for Ingress to the OpenShift Router Pods. We can replace it with Company signed Wildcard Certificate.

This certificate will be used if SSL is going to be terminated on the OpenShift Routers (default option called `Edge` in OpenShift). 

Since our certificate is not signed by Public authority and contains IntermediateCA, we will present it together with Certificate forming the chain.

```bash
|- wildcard certificate
  |- intermediateCA
    |- RootCA
```

RootCA will be created as part of trustedCA bundle and controlled by OpenShift Global Proxy configuration.


## Ingress Certificate Role <a name="paragraph1"></a>

```bash
certificates/
├── README.md
├── defaults
│   └── main.yml
├── files
│   └── ca-bundle.crt
├── images
│   └── diagram.svg
├── tasks
│   └── main.yml
└── vars
    ├── main.yml
    └── vault.yml
```

Role contains `files` directory where we place our `ca-bundle.crt` file (usually taken from one of the corporate VMs `/etc/pki/tls/certs/` directory since corporate VMs has company signed certificates in thier bundle)

We use Ansible Vault for Ingress Certificate and Private Key.

`vars/vault.yml` holds certificate and private_key key:value pairs:

```yaml
certificate: |
  -----BEGIN CERTIFICATE-----
  <encrypted certificate data>
  -----END CERTIFICATE-----
  subject=/DC=tv/DC=ebms/CN=EBMS Issuing CA 1
  issuer=/O=EBMS/CN=EBMS Root Certificate Authority
  -----BEGIN CERTIFICATE-----
  <encrypted certificate data>
  -----END CERTIFICATE-----
private_key: |
  -----BEGIN RSA PRIVATE KEY-----
  <encrypted private key data>
  -----END RSA PRIVATE KEY-----
```


In order to create vault.yml file, run `ansible-vault create vault.yml`, specify the password. To edit the vault file, run `ansible-vault edit vault.yml`.

When running ansible-playbook, pass the `--ask-vault-pass` obtion.

Role performs these tasks:

* *Remove old Config Map for CA Bundle if exists*
* *Create Config Map for CA Bundle* - we are giving ConfigMap the name - custom-ca

:exclamation: We are recreating ca-bundle ConfigMap because ConfigMaps cannot be patched.

* *Patch Global Proxy with new trustedCA bundle* - Here we are setting new ca-bundle as trusted
* *Create a Secret with crt and key* - Here we are creating a Kubernetes Secret object containing our Centrificate and Private Key
* *Patch Ingress Controller* - Here we tell Ingress Controller to use our certificate Secret as a default Ingress certificate

```yaml
---
- name: Remove old Config Map for CA Bundle if exists 
  k8s:
    state: absent
    kind: ConfigMap
    name: custom-ca
    namespace: openshift-config

- name: Create Config Map for CA Bundle
  k8s:
    state: present
    kind: ConfigMap
    name: custom-ca
    namespace: openshift-config
    definition:
      data: 
        ca-bundle.crt: "{{ lookup('file', role_path + '/files/ca-bundle.crt') }}" 
  ignore_errors: true

# Should perform a patch on the proxy/cluster object
- name: Patch Global Proxy with new trustedCA bundle
  k8s:
    state: present
    merge_type: 
      - strategic-merge
      - merge
    kind: Proxy
    name: cluster
    definition:
      spec:
        trustedCA:
          name: custom-ca

- name: import certificate vars
  include_vars:
    file: "{{ role_path }}/vars/vault.yml"

- name: Create a Secret with crt and key
  ignore_errors: true
  k8s:
    state: present
    kind: secret
    name: '{{ ingress_certificate_name }}'
    namespace: openshift-ingress
    definition:
      data:
        tls.crt: "{{ certificate | b64encode }}"
        tls.key: "{{ private_key | b64encode }}"

- name: Patch Ingress Controller
  k8s:
    state: present
    merge_type: 
      - strategic-merge
      - merge
    kind: IngressController
    name: default
    namespace: openshift-ingress-operator
    definition:
      spec:
        defaultCertificate:
          name: '{{ ingress_certificate_name }}'
```

:exclamation: If Ingress Certificate expiration day is aproaching, update the `group_vars/all` providing new Ingress Certificate name `ingress_certificate_name`. Good practice is to keep the same name and only add expiry data as suffix (e.g. ingress-certificate-20220330) so it can be updated without downtime. 
Once the Ingress Certificate is updated, login to OpenShift and remove the old secret:

```yaml
oc delete secret ingress-certificate -n openshift-ingress
```

