# DEPREACTO
Con keycloak 26.x, il discovery fra le varie repliche di keycloak avviene mediante JDBC_PING.
Sotto trovi comunque la spiegazione del perché usare KUBE_PING, RBAC etc.

Questo era lo yaml per configurare KUBE_PING:
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: keycloak-sa
  namespace: keycloak
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: keycloak-role
  namespace: keycloak
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: keycloak-rb
  namespace: keycloak
subjects:
  - kind: ServiceAccount
    name: keycloak-sa
roleRef:
  kind: Role
  name: keycloak-role
  apiGroup: rbac.authorization.k8s.io
```

# RBAC per KUBE_PING invece di un Headless Service

Keycloak usa:
- Infinispan embedded
- JGroups
- discovery via DNS

Esistono due modalità di discovery per JGroups su Kubernetes:

### A) DNS_PING (quella della headless service)
- usa DNS
- NON richiede permessi API
- usa -Djgroups.dns.query=...

### B) KUBE_PING (via Kubernetes API)
Qui JGroups:
  - chiama l’API server
  - fa GET /api/v1/pods
  - legge i pod con label matching
  - costruisce il cluster

Per fare questo servono permessi RBAC:
```yaml
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
```

Senza questi permessi:
- JGroups non vede gli altri pod
- cluster view = solo sé stesso
- cache non distribuita

## Con Headless Service ogni pod deve conoscere lo IP degli altri POD e questa configurazione sarebbe critica perche':
- Headless Service ha clusterIP: None
- Una service normale restituisce un solo ClusterIP
- Una headless service restituisce tutti gli IP dei pod
- JGroups fa una query DNS tipo:

Headless Service si aspetta una lista di IP.

Se:
- il nome DNS è sbagliato
- il service selector non matcha i pod
- la porta jgroups non è esposta
- la headless non esiste

Allora:
👉 il cluster non si forma
👉 ogni nodo resta standalone
👉 la cache è locale
👉 login random → logout casuali → sessioni perse

È il punto più fragile del sistema.

## RBAC + KUBE_PING
KUBE_PING fa:
1. Chiamata all’API Kubernetes
2. Lista i pod nel namespace
3. Filtra per label
4. Costruisce la cluster view

In pratica esegue qualcosa di equivalente a:
```
GET /api/v1/namespaces/keycloak/pods
```

Se il Pod non ha permessi:
- HTTP 403
- cluster view = solo sé stesso 
- cache locale

In Kubernetes RBAC abbiamo 4 oggetti principali:

| Oggetto        | Cosa fa                         |
|----------------|---------------------------------|
| ServiceAccount | Identità del Pod                |
| Role           | Permessi dentro un namespace    |
| ClusterRole    | Permessi cluster-wide           |
| RoleBinding    | Collega Role <-> ServiceAccount |

Nel nostro caso:
- Ci serve accesso solo nel namespace keycloak
- Quindi basta una Role
- Non serve ClusterRole