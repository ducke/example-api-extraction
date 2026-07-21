# Spike: vendor-neutrale IaaS-API via resource-broker (floci-gcp + floci-aws, kind)

**Status: Geruest — noch nicht ausgefuehrt/verifiziert.** Zum Durchklicken beim
IPCEI-Hackathon (Potsdam, 21.–24.07.2026, Topic AE09 „Standardized IaaS APIs").

## Was der Spike zeigt

**Ein** generischer Typ `storage.generic.platform-mesh.io/Object` (Bucket),
**zwei** Provider (GCP → floci-gcp, AWS → floci-aws), und der
[resource-broker](https://github.com/platform-mesh/platform-mesh/tree/main/operators/resource-broker)
routet die Bestellung nach `region` — inklusive Live-**Migration** beim
Region-Wechsel. Das ist die AE09-These zum Anfassen: eine Bestellung, zwei
Clouds, Provider-Wechsel ohne Ticket.

```
Consumer (root:consumer)
  Object{region: eu}                      ┌── AcceptAPI region=eu ──▶ GCP-Provider
        │  bindet generische objects-API  │      kro-RGD → Job → floci-gcp (gs://…)
        ▼                                 │
  resource-broker  ── routet nach region ─┤
   (root:platform, Staging/Assignment)    │
                                          └── AcceptAPI region=us ──▶ AWS-Provider
   region eu→us  ⇒  Migration, Cutover           kro-RGD → Job → floci-aws (s3://…)
```

Der Baustein darunter — Crossplane/provider-terraform gegen einen einzelnen
floci-Emulator — wurde in einem vorangegangenen GCP-Spike verifiziert (ein
Provider, fest ueber api-syncagent verdrahtet). Dieses Beispiel setzt die
**Routing-Schicht** darueber (Broker) und einen **zweiten** Provider obendrauf;
die Realisierung laeuft hier ueber **kro** (Upstream-Muster), nicht Crossplane.

## Architektur-Entscheidungen (festgelegt)

- **Realisierung: kro RGD**, nicht Crossplane — wie die Upstream-Beispiele
  (`broker-certificates`, `broker-postgres`). kro erzeugt aus der RGD selbst die
  Provider-CRD und relayed auf ein Kubernetes-Workload.
- **kro-Ziel: ein Job mit CLI** (`curl` gegen floci-gcp, `aws` gegen floci-aws) —
  garantiert lokal lauffaehig, null Crossplane. Analog zum `kropg`-Provider
  (Postgres als schlichtes Deployment). ACK/Config-Connector mit
  Endpoint-Override waeren die „so saehe es echt aus"-Ausbaustufe.
- **Topologie: ein kind-Cluster.** Die Beispiele nutzen drei (platform + 2
  Provider); hier kollabiert auf einen Node — Broker + kro + beide floci +
  2× syncagent. kcp-Workspaces bleiben logisch getrennt.

## Verzeichnis

```
manifests/floci-gcp.yaml          Emulator GCS  (:4588)
manifests/floci-aws.yaml          Emulator S3   (:4566)   [VERIFY Image/Port]
platform/README.md                die zwei Platform-APIExports (Haupt-Handverdrahtung)
providers/gcp/acceptapi.yaml      region=eu  ──┐
providers/gcp/rgd-object.yaml     kro → floci-gcp
providers/gcp/publishedresource-objects.yaml   syncagent → root:providers:gcp
providers/aws/…                   dito, region=us → floci-aws
consumer/apibinding-objects.yaml  Consumer bindet generische API
consumer/order-object.yaml        die Bestellung + Migrations-Patch (im Kommentar)
tasks/todo.md                     Reihenfolge, offene Punkte, Risiken
```

## Drehbuch

### 0. Prereqs

`docker`, `kind`, `kubectl`, `helm`, `yq`, `go`, sowie die
[kcp-kubectl-Plugins](https://docs.kcp.io/kcp/main/setup/kubectl-plugin/)
(per krew; `export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"`).

resource-broker-Repo sparse auschecken:

```bash
git clone --depth 1 --filter=blob:none --sparse \
  https://github.com/platform-mesh/platform-mesh.git pm
cd pm && git sparse-checkout set operators/resource-broker
cd operators/resource-broker
source ./hack/lib.bash    # helm::install::* Helper
```

### 1. Ein kind-Cluster + Basis

Die Beispiel-`run.bash` baut drei Cluster; wir nehmen einen und richten die Basis
mit denselben Helpern ein (alles gegen EINEN Kubeconfig):

```bash
kind create cluster --name broker-poc
KC=~/.kube/config
helm::install::certmanager "$KC"
helm::install::etcddruid   "$KC"
helm::install::kcp         "$KC"
helm::install::kro         "$KC"
```

> Falls die Helper Cluster-spezifische Namen erwarten: als Vorlage `_setup()` /
> `_provider_setup()` in `examples/broker-postgres/run.bash` lesen und die
> `kind create`-Zeilen auf einen Cluster reduzieren.

### 2. kcp-Workspaces + Platform-Exports

Struktur aus `broker-postgres/run.bash _setup` uebernehmen:

```
root:platform                      (+ AcceptAPI-Export, + generischer objects-Export)
root:platform:broker               (Assignment/Migration/StagingWorkspace)
root:platform:broker:staging
root:platform:broker:verification
root:providers:gcp                 (Typ universal, manuelles APIBinding)
root:providers:aws
root:consumer
```

Den **generischen `objects`-Export** bereitstellen → siehe `platform/README.md`
(die einzige groessere Handverdrahtung: CRD→ARS→APIExport). Alle Workspaces vom
Typ `universal` mit manuellem APIBinding — **nie** `org`/`account` direkt anlegen
(Lektion aus einem frueheren kcp-Spike: sonst haengen sie im
`root:security`-Initializer).

### 3. Broker starten

```bash
./examples/broker-postgres/run.bash start-broker   # baut & startet den Broker im Cluster
```

### 4. Provider einrichten (GCP & AWS)

Fuer beide Provider (`gcp`, `aws`) — Kubeconfig zeigt hier auf denselben Cluster,
nur der Ziel-Workspace unterscheidet sich:

```bash
# Emulatoren
kubectl apply -f manifests/floci-gcp.yaml
kubectl apply -f manifests/floci-aws.yaml

# je Provider: kro-RGD (erzeugt die Object-CRD) + syncagent-PublishedResource
kubectl apply -f providers/gcp/rgd-object.yaml
kubectl apply -f providers/aws/rgd-object.yaml
# api-syncagent je Provider installieren (helm::install::api_syncagent … "objects" …),
# dann:
kubectl apply -f providers/gcp/publishedresource-objects.yaml
kubectl apply -f providers/aws/publishedresource-objects.yaml

# je Provider: AcceptAPI-Export binden, dann AcceptAPI anlegen
#   (APIBinding acceptapis im jeweiligen Provider-Workspace — Manifest wie im
#    Postgres-Beispiel, permissionClaims: secrets get/list/watch)
kubectl --context …:providers:gcp apply -f providers/gcp/acceptapi.yaml
kubectl --context …:providers:aws apply -f providers/aws/acceptapi.yaml

# beide AcceptAPIs muessen Ready werden (Broker verifiziert Bindbarkeit):
kubectl wait acceptapi/objects.storage.generic.platform-mesh.io --for=condition=Ready --timeout=5m
```

### 5. Bestellen (Consumer)

```bash
kubectl --context …:consumer apply -f consumer/apibinding-objects.yaml
kubectl --context …:consumer wait apibinding/objects --for=condition=Ready --timeout=5m
kubectl --context …:consumer apply -f consumer/order-object.yaml
```

Erwartung: Broker legt ein `Assignment` an (→ GCP), staged das Object, syncagent
zieht es auf den Compute-Cluster, kro-Job legt `gs://bucket-from-consumer` in
floci-gcp an, `status.url` erscheint im Consumer-Workspace.

Gegenprobe am Emulator:

```bash
kubectl -n floci-gcp run verify --rm -i --restart=Never --image=curlimages/curl -- \
  -s 'http://floci-gcp.floci-gcp.svc.cluster.local:4588/storage/v1/b?project=test-project'
```

### 6. Migration eu → us (der Hoehepunkt)

```bash
kubectl --context …:consumer patch object bucket-from-consumer \
  --type merge -p '{"spec":{"region":"us"}}'
```

Broker erzeugt eine `Migration`, staged beim AWS-Provider, beide bedienen kurz
parallel, dann Cutover: `s3://bucket-from-consumer` in floci-aws, GCP-Kopie wird
abgeraeumt, `status.url` zeigt jetzt auf S3.

```bash
kubectl -n floci-aws run verify --rm -i --restart=Never --image=amazon/aws-cli:2.31.14 \
  --env AWS_ACCESS_KEY_ID=test --env AWS_SECRET_ACCESS_KEY=test -- \
  --endpoint-url http://floci-aws.floci-aws.svc.cluster.local:4566 s3api list-buckets
```

### Aufraeumen

```bash
kind delete cluster --name broker-poc
```

## Bekannte Grenzen / Risiken

Siehe `tasks/todo.md`. Kurz: create-only-Jobs (kein Bucket-Delete beim Cutover;
floci ist ephemer), floci-aws-Image/Port/Env noch zu verifizieren, und der
generische `objects`-APIExport ist die einzige nennenswerte Handverdrahtung.
