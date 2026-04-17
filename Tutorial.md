# Config Sync Playground: RootSync und RepoSync

Dieses Repository enthaelt ein minimales Setup, mit dem du RootSync und RepoSync auf einem lokalen Kind-Cluster vergleichen kannst. Die Config Sync Installation selbst wird hier nicht beschrieben.

## Zielbild

Config Sync liest nicht dein lokales Arbeitsverzeichnis, sondern eine Git-, OCI- oder Helm-Quelle. In diesem Playground verwenden beide Sync-Typen dieselbe Git-Quelle, aber unterschiedliche Verzeichnisse:

| Sync-Typ | Quelle | Zweck |
| --- | --- | --- |
| RootSync | `root-sync/` | Plattform- bzw. clusterweite Konfiguration |
| RepoSync | `repo-sync/team-a/` | Namespace-Konfiguration fuer ein Team oder eine App |
| Bootstrap | `bootstrap/` | Manifeste, die du einmalig mit `kubectl apply` anlegst |

Die Bootstrap-Manifeste zeigen aktuell auf:

```text
https://github.com/adilosch/configsync.git
```

Wenn dein Remote anders heisst oder das Repository privat ist, passe `spec.git.repo` in diesen Dateien an:

- `bootstrap/10-root-sync.yaml`
- `bootstrap/20-repo-sync-team-a.yaml`

Fuer `auth: none` muss die HTTPS-URL aus dem Kind-Cluster heraus ohne Credentials lesbar sein. Fuer private Repositories brauchst du spaeter eine Config Sync Auth-Variante, zum Beispiel SSH oder Token. Fuer dieses Lernsetup ist ein oeffentlich lesbares Test-Repository am einfachsten.

## Struktur

```text
bootstrap/
  00-team-a-namespace.yaml
  10-root-sync.yaml
  20-repo-sync-team-a.yaml

root-sync/
  cluster/
    root-demo-namespace.yaml
    root-demo-reader-clusterrole.yaml
  namespaces/root-demo/
    default-deny-networkpolicy.yaml
    platform-settings.yaml

repo-sync/team-a/
  app-config.yaml
  app-reader-role.yaml
  app-reader-rolebinding.yaml
  app-serviceaccount.yaml
```

## 1. Aenderungen in Git verfuegbar machen

Config Sync synchronisiert aus Git. Deshalb muessen die Dateien in dem Branch liegen, den `RootSync` und `RepoSync` referenzieren.

```bash
git status
git add bootstrap root-sync repo-sync Tutorial.md
git commit -m "Add Config Sync RootSync and RepoSync playground"
git push origin main
```

Wenn du einen anderen Branch verwenden willst, passe `spec.git.branch` in den beiden Bootstrap-Dateien an.

## 2. Bootstrap-Ressourcen anlegen

Pruefe zuerst, dass du auf deinem Kind-Cluster bist:

```bash
kubectl config current-context
```

Lege dann den Namespace fuer `RepoSync` und beide Sync-Objekte an:

```bash
kubectl apply -f bootstrap/
```

Das erzeugt:

- Namespace `team-a`
- `RootSync/root-sync-playground` im Namespace `config-management-system`
- `RepoSync/repo-sync-team-a` im Namespace `team-a`

## 3. Sync-Status beobachten

Wenn `nomos` installiert ist:

```bash
nomos status
```

Alternativ mit `kubectl`:

```bash
kubectl get rootsync -n config-management-system root-sync-playground
kubectl get reposync -n team-a repo-sync-team-a
kubectl describe rootsync -n config-management-system root-sync-playground
kubectl describe reposync -n team-a repo-sync-team-a
```

Die Reconciler-Pods findest du hier:

```bash
kubectl get pods -n config-management-system
```

## 4. Ergebnis von RootSync pruefen

RootSync darf clusterweite Ressourcen und Ressourcen in mehreren Namespaces verwalten.

```bash
kubectl get namespace root-demo --show-labels
kubectl get clusterrole root-demo-configmap-reader
kubectl get configmap -n root-demo platform-settings -o yaml
kubectl get networkpolicy -n root-demo default-deny-ingress
```

Wichtig zum Verstaendnis:

- `root-demo` wurde aus `root-sync/cluster/root-demo-namespace.yaml` erzeugt.
- `root-demo-configmap-reader` ist ein `ClusterRole` und damit cluster-scoped.
- Die ConfigMap und NetworkPolicy liegen im Namespace `root-demo`, werden aber trotzdem vom RootSync verwaltet.

Das entspricht grob einem Argo CD Application-Modell fuer Plattform-Konfiguration mit breitem Scope.

## 5. Ergebnis von RepoSync pruefen

RepoSync ist namespaced. Das Sync-Objekt liegt in `team-a` und synchronisiert die App- bzw. Team-Ressourcen fuer diesen Namespace.

```bash
kubectl get configmap -n team-a app-config -o yaml
kubectl get serviceaccount -n team-a demo-app
kubectl get role -n team-a app-config-reader
kubectl get rolebinding -n team-a demo-app-reads-config
```

Wichtig zum Verstaendnis:

- `RepoSync` ist ideal fuer tenant- oder teamnahe Konfiguration.
- Dieses Beispiel erzeugt nur namespaced Ressourcen.
- Ein `RepoSync` sollte nicht fuer clusterweite Plattform-Objekte verwendet werden.

Das entspricht gedanklich eher einer Argo CD Application, die einem Team-Namespace gehoert und nicht den ganzen Cluster modelliert.

## 6. Drift ausprobieren

Aendere eine verwaltete Ressource direkt im Cluster:

```bash
kubectl patch configmap -n team-a app-config --type merge -p '{"data":{"owner":"manual-change"}}'
kubectl get configmap -n team-a app-config -o jsonpath='{.data.owner}{"\n"}'
```

Warte einige Sekunden und pruefe erneut:

```bash
kubectl get configmap -n team-a app-config -o jsonpath='{.data.owner}{"\n"}'
```

Config Sync sollte den Wert wieder auf `team-a` zuruecksetzen, weil Git die gewuenschte Quelle ist.

Dasselbe fuer RootSync:

```bash
kubectl patch configmap -n root-demo platform-settings --type merge -p '{"data":{"owner":"manual-change"}}'
kubectl get configmap -n root-demo platform-settings -o jsonpath='{.data.owner}{"\n"}'
```

Nach der naechsten Reconciliation sollte wieder `platform-team` erscheinen.

## 7. Git-Aenderung ausprobieren

Aendere in `repo-sync/team-a/app-config.yaml` zum Beispiel:

```yaml
data:
  owner: team-a-platform
```

Dann committen und pushen:

```bash
git add repo-sync/team-a/app-config.yaml
git commit -m "Change team-a app owner"
git push origin main
```

Nach kurzer Zeit sollte der neue Wert im Cluster erscheinen:

```bash
kubectl get configmap -n team-a app-config -o jsonpath='{.data.owner}{"\n"}'
```

## 8. Scope-Unterschied praktisch sehen

RootSync verwaltet in diesem Beispiel:

```bash
kubectl get all,configmap,networkpolicy -n root-demo
kubectl get clusterrole root-demo-configmap-reader
```

RepoSync verwaltet in diesem Beispiel:

```bash
kubectl get all,configmap,role,rolebinding,serviceaccount -n team-a
```

Als Merksatz:

- RootSync ist fuer Cluster- und Plattform-Baseline.
- RepoSync ist fuer Namespace- oder Team-Konfiguration.
- Beide koennen aus demselben Git-Repository lesen, sollten aber unterschiedliche Verzeichnisse und Verantwortlichkeiten haben.

## 9. Aufraeumen

Zuerst die Sync-Objekte entfernen:

```bash
kubectl delete -f bootstrap/
```

Dann die von Config Sync erzeugten Demo-Ressourcen entfernen:

```bash
kubectl delete namespace root-demo
kubectl delete clusterrole root-demo-configmap-reader
kubectl delete namespace team-a
```

Wenn Config Sync noch laeuft, entferne zuerst die Sync-Objekte, sonst kann es geloeschte Ressourcen wiederherstellen.

## Troubleshooting

Pruefe bei Fehlern zuerst die Sync-Objekte:

```bash
kubectl describe rootsync -n config-management-system root-sync-playground
kubectl describe reposync -n team-a repo-sync-team-a
```

Typische Ursachen:

- Das Git-Repository ist aus dem Kind-Cluster nicht erreichbar.
- `auth: none` wird verwendet, aber das Repository ist privat.
- Der Branch in `spec.git.branch` existiert nicht.
- Das Verzeichnis in `spec.git.dir` existiert im Remote-Branch noch nicht.
- Die Aenderungen wurden lokal erstellt, aber noch nicht gepusht.
