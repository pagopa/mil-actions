# Trigger Remote Workflow

Composite GitHub Action che:

1. avvia un workflow in un repository remoto,
2. attende che GitHub registri la nuova esecuzione,
3. monitora lo stato della run fino al completamento,
4. fallisce se la run remota termina con esito diverso da `success` o va in timeout.

## Come funziona

L'action usa `gh` CLI con questi comandi:

- `gh workflow run` per avviare il workflow remoto;
- `gh run list` per recuperare l'ID della run piu recente;
- `gh run view` in polling per leggere `status` e `conclusion`.

Supporta anche input dinamici verso il workflow remoto tramite `workflow_inputs` nel formato:

```text
key1=value1,key2=value2
```

che vengono convertiti in:

```text
--raw-field key1=value1 --raw-field key2=value2
```

## Input

| Nome | Obbligatorio | Default | Descrizione |
| --- | --- | --- | --- |
| `remote_gh_token` | Sì | - | Token GitHub usato dalla `gh` CLI verso il repository remoto. |
| `remote_repo` | Sì | - | Repository remoto nel formato `owner/repo`. |
| `remote_workflow` | Sì | - | File workflow da avviare nel repository remoto (es. `deploy.yml`). |
| `waiting_after_trigger` | No | `10` | Secondi di attesa dopo il trigger prima di leggere la run. |
| `waiting_between_attempts` | No | `20` | Secondi tra un controllo stato e il successivo. |
| `max_attempts` | No | `30` | Numero massimo di tentativi prima del timeout. |
| `workflow_inputs` | No | `""` | Lista `key=value` separata da virgola da passare al workflow remoto. |

## Permessi richiesti per `remote_gh_token`

Il token deve poter chiamare le API GitHub Actions sul **repository remoto** (`remote_repo`) per:

- avviare una run (`workflow_dispatch`),
- leggere lista run,
- leggere stato e conclusione run.

Configurazione consigliata:

- **Fine-grained PAT**:
  - repository access: almeno il repository indicato in `remote_repo`;
  - repository permission `Actions`: **Read and write**.
- **Classic PAT** (se usato):
  - scope `repo` (o `public_repo` se il repository remoto e pubblico).

> Nota: in alcune organizzazioni possono esserci policy aggiuntive che richiedono scope/permessi ulteriori.

## Esempio d'uso

```yaml
jobs:
  trigger:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Trigger workflow remoto
        uses: pagopa/mil-actions/trigger-remote-workflow@main
        with:
          remote_gh_token: ${{ secrets.REMOTE_GH_TOKEN }}
          remote_repo: pagopa/idpay-deploy-aks
          remote_workflow: deploy.yml
          waiting_after_trigger: "15"
          waiting_between_attempts: "20"
          max_attempts: "40"
          workflow_inputs: "environment=uat,version=1.2.3"
```

## Comportamento di fallimento

L'action termina con errore se:

- il trigger del workflow remoto fallisce;
- viene raggiunto `max_attempts` prima del completamento;
- la run remota termina con `conclusion != success`.
