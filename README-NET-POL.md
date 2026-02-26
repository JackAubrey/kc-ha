# Network Policy

Quando NON esiste nessuna NetworkPolicy:

👉 Tutti i Pod possono parlare con tutti.

Quando crei una NetworkPolicy che seleziona un Pod:

👉 Quel Pod entra in modalità “whitelist”.

Significa:
- Tutto il traffico è bloccato
- Tranne quello esplicitamente permesso

MA solo per la direzione dichiarata.

Ad esempio se scriviamo:
```yaml
policyTypes:
  - Ingress
```

Stiamo dicendo: Blocca tutto il traffico in ingresso, salvo eccezioni. La egress resta aperta.

```yaml
podSelector: {}
policyTypes:
  - Ingress
```

Se lasciassimo vuoto il selettore significherebbe = tutti i pod nel namespace.

Risultato:

👉 Tutti i pod del namespace keycloak non accettano più traffico in ingresso.

Quindi bloccheremo il traffico in entrata verso il solo keycloak.

Nel nostro caso specifico di [net-pol-deny-keycloak.yaml](templates/infra/net-pol-deny-keycloak.yaml) andremo a bloccare il traffico in entrata verso keycloak.

Ora serve aprire esplicitamente creando una Network Policy per:
- di tipo ingress per lo ingress controller: [net-pol-allow-ingress-controller.yaml](templates/infra/net-pol-allow-ingress-controller.yaml)
- di tipo ingress per un ipotetico back-end e front-end
- di tipo ingress per per il DB
- di tipo egress per il DB. Questo fa si che keycloak possa andare solo sul DB e da nessun'altra parte.

Se non esiste una Network Policy per una direzione, di default tutto il traffico è autorizzato.

| Direzione | Esiste policy per quella direzione? | Comportamento  |
| --------- | ----------------------------------- | -------------- |
| Ingress   | No                                  | Tutto permesso |
| Ingress   | Sì                                  | Default deny   |
| Egress    | No                                  | Tutto permesso |
| Egress    | Sì                                  | Default deny   |
