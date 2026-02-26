# Pod Disruption Budget

Protegge da:
- drain nodo
- rolling upgrade
- eviction

Un PodDisruptionBudget dice a Kubernetes: “Durante una disruption volontaria, quanti Pod devono rimanere disponibili?”

⚠️ Parliamo SOLO di:
- kubectl drain
- upgrade nodo
- rolling update
- eviction volontarie

NON protegge da:
- crash del nodo
- OOM (Out Of Memory)
- kill del container

# minAvailable = N-1
È consigliabile impostare il valore minAvailable uguale al numero di **repliche** dei POD meno 1 perchè PDB lavora sui Pod selezionati.

Esempio:

Hai:
- 3 nodi
- 2 repliche di Keycloak
- PDB: minAvailable: 1

Significa che Kubernetes può rendere non disponibili al massimo 1 Pod alla volta.

Il PDB risponde alla domanda: **Durante manutenzione, quante repliche voglio garantire vive?**