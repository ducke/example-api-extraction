# TODO — resource-broker Bucket-Spike (floci-gcp + floci-aws)

## Vor dem Hackathon (jetzt billig, vor Ort teuer)

- [ ] **floci-aws verifizieren**: Image-Tag, Port (4566?), Env-Praefix
      (`FLOCI_AWS_*`?) und ein `s3api create-bucket` gegen `:4566` lokal testen
      (Docker, ohne kind). Quelle: https://floci.io/aws/ · https://github.com/floci-io/floci
- [ ] **kcp-kubectl-Plugins** per krew installieren, PATH setzen.
- [ ] **resource-broker-Image**: `start-broker` baut aus dem Repo (Go noetig) —
      Build vorab durchziehen, Image lokal cachen. (Upstream-TODO „use prebuilt
      image" ist noch offen.)
- [ ] **Generischer `objects`-APIExport**: CRD→ARS→APIExport vorbereiten
      (CRD->ARS-Konvertierung, mechanisch). Siehe platform/README.md.

## Setup-Reihenfolge (Kurzform)

1. [ ] kind-Cluster `broker-poc` + Basis (cert-manager, etcd-druid, kcp, kro)
2. [ ] kcp-Workspaces + Broker-State-Workspaces + beide Platform-Exports
3. [ ] `start-broker`
4. [ ] floci-gcp + floci-aws deployen
5. [ ] je Provider: RGD, syncagent + PublishedResource, AcceptAPI-Binding, AcceptAPI
6. [ ] beide AcceptAPIs Ready
7. [ ] Consumer: objects binden, Object{region:eu} bestellen → floci-gcp
8. [ ] Migration: region→us patchen → floci-aws, GCP abgeraeumt

## Offene technische Punkte

- [ ] **Single-Cluster vs. run.bash**: `run.bash` ist auf 3 Cluster ausgelegt.
      Entweder `_setup`/`_provider_setup` auf einen Cluster anpassen, oder die
      Helper aus `hack/lib.bash` einzeln gegen einen Kubeconfig fahren. Pruefen,
      ob der syncagent zwei Instanzen (gcp/aws) auf demselben Compute-Cluster
      sauber trennt (unterschiedliche agent-name/Namespace).
- [ ] **kro CEL `.orValue(...)`**: Syntax gegen die eingesetzte kro-Version
      verifizieren; sonst auf `has(...) ? … : …` umstellen.
- [ ] **status.url aus Job-Annotation**: Muster (Annotation aus schema stempeln,
      in status lesen) am kropg-Beispiel bestaetigt — bei leerer Annotation
      Fallback pruefen.
- [ ] **RGD-Schema ↔ generische CRD**: `spec{region,versioning}` muss zur
      Upstream-Object-CRD passen, sonst validiert die Broker-Kopie nicht.

## Bewusste Grenzen (PoC, nicht produktiv)

- **create-only**: Jobs legen Buckets an; Loeschen/Cutover raeumt nur den Job,
  nicht den Bucket. floci ist ephemer (kind delete). Produktiv: Delete-Job +
  Finalizer in der RGD.
- **Kein Portal**: Broker braucht keins — spart die Portal-Stolpersteine
  (die Portal-spezifischen Fallen des GCP-Spikes entfallen). Bestellung per kubectl.
- **Auth fake**: floci ignoriert Creds grundsaetzlich.

## Ausbaustufen (wenn Zeit bleibt)

- [ ] kro-Ziel von Job → ACK (`s3…/Bucket`) + Config-Connector (`StorageBucket`)
      mit Endpoint-Override auf floci — „echte" Provider-Controller.
- [ ] MigrationConfiguration mit Zwischenstufe (Datentransfer), wie im
      Postgres-Beispiel, statt direktem Cutover.
- [ ] Zweiter generischer Typ (z. B. `compute/KubernetesCluster` → Gardener)
      als Beleg, dass das Muster ueber Buckets hinaus traegt.
