# charts

#INSTALAR VAULT

helm repo add hashicorp https://helm.releases.hashicorp.com               
helm repo update
helm install vault hashicorp/vault

#INSTALAR EXTERNAL SECRETS

helm repo add external-secrets https://charts.external-secrets.io         
helm install external-secrets external-secrets/external-secrets

#INSTALAR CLI VAULT

curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com focal main"    
sudo apt-get update
sudo apt-get install vault

#AGREGAR INGRESS VAULT

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: vault-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
    - host: vault.craftech.io
      http:
        paths:
          - path: /()(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: vault
                port:
                  number: 8200



k apply -f vault-ingress.yml  

#CONFIGURAR INGRESS

sudo nano /etc/hosts

Agregar una entrada para nuestros ingress

192.168.49.2 integrador.craftech.io
192.168.49.2 vault.craftech.io

Chequear que la IP sea la correspondiente a minikube ip
    

#CONFIGURAR VAULT UI

Ingresar mediante el ingress, configurar el token y key, guardarlos en un lugar seguro

#CONFIGURAR VAULT CLI

export VAULT_ADDR="http://vault.craftech.io"

vault login 

Ingresar el token

#CREAR SECRET VAULT

vault secrets enable -path=secrets kv-v2
vault kv put secrets/postgres postgres-user="postgres" postgres-password="example"


#AGREGAR CLUSTER SECRET STORE

echo -n “nuestro_vault_token" | base64 #ESTO ES EL VAULT_TOKEN_BASE_64, VA EN EL SECRET DEL SECRET CLUSTER STORE

apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: vault-cluster-secret-store
  namespace: default
spec:
  provider:
    vault:
      server: "http://vault.craftech.io/"
      path: "secrets"
      version: "v2"
      auth:
        tokenSecretRef:
          name: "vault-token"
          key: "token"
---
apiVersion: v1
kind: Secret
metadata:
  name: vault-token
  namespace: default
type: Opaque
data:
  token: VAULT_TOKEN_BASE_64


k apply -f secret-cluster-store.yml


#AGREGAR EXTERNAL SECRET

apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: vault-external-secret
  namespace: default
spec:
  refreshInterval: "15s"
  secretStoreRef:
    name: vault-cluster-secret-store
    kind: ClusterSecretStore
  target:
    name: vault-secret
    creationPolicy: Owner
  data:
  - secretKey: postgres-user
    remoteRef:
      key: secrets/postgres
      property: POSTGRES_USER  
  - secretKey: postgres-password
    remoteRef:
      key: secrets/postgres
      property: POSTGRES_PASSWORD


k apply -f external-secret.yml

