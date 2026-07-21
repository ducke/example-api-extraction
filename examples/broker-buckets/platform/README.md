# Platform-Workspace: die zwei APIExports

Der resource-broker braucht im Platform-Workspace (`root:platform`) zwei
APIExports. Beide werden in den Upstream-Beispielen von `run.bash setup`
programmatisch erzeugt — fuer diesen Bucket-Spike musst du sie einmal
bereitstellen:

## 1. AcceptAPI-Export (fuer Provider)

Kommt fertig aus dem resource-broker-Repo:
`config/broker/crd/broker.platform-mesh.io_acceptapis.yaml`.
Provider binden diesen Export, um ihre `AcceptAPI` anzulegen.

## 2. Generischer `objects`-Export (fuer Consumer)

Die vom Broker "backed" API, die der Consumer bindet und bestellt.
Basis-CRD liegt upstream: `config/generic/crd/storage.generic.platform-mesh.io_objects.yaml`.

kcp will kein CRD, sondern eine **APIResourceSchema** + **APIExport**. Die
CRD->ARS-Konvertierung ist mechanisch (die Beispiele im resource-broker-Repo
erzeugen sie in `run.bash setup`; ein CRD->ARS-Skript aus einem frueheren
GCP-Spike lag ebenfalls vor). Ergebnis:

```
APIResourceSchema  v1alpha1.objects.storage.generic.platform-mesh.io
APIExport          objects   (spec.resources[].schema -> obige ARS)
```

> HAUPT-TODO vor Ort: Diese ARS + den `objects`-APIExport erzeugen. Das ist die
> einzige groessere Handverdrahtung; alles andere (Broker, kro, syncagent) kommt
> aus den Repo-Helpern bzw. den Manifesten in `providers/` und `consumer/`.

## Broker-State-Workspaces

`root:platform:broker` (haelt `Assignment`/`Migration`/`StagingWorkspace`) mit
Kindern `staging` und `verification`. Ebenfalls von `run.bash setup` angelegt —
Struktur 1:1 aus dem Postgres-Beispiel uebernehmen.
