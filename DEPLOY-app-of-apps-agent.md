# Wdrożenie: app-of-apps na hubie z argocd-agent (sync-wave honorowany)

Cel: parent `Application` jest reconcilowany przez application-controller **na hubie**, tworzy dzieci falami
(`app21` fala 0, `app22` fala 1) i czeka, aż każda fala będzie `Healthy`. Dzieci trafiają do spoke'a przez
principal i agenta (`spec.destination.name`).

```
PipelineRun -> argocd-parent -> Application "tenant0-spoke1-tenant0" (hub, destination: hub)
hub application-controller -> fala 0: app21 (label argocd-agent=true) -> principal -> agent -> spoke
                           <- czeka na status Healthy dziecka (zwraca go principal)
                           -> fala 1: app22 -> ... -> spoke
argocd-wait czeka na parenta
```

Stary chart `app-of-apps/` i ApplicationSet nie są zmieniane. Nowy chart to `app-of-apps-agent/`.

Wszystko oznaczone **[HIPOTEZA]** nie było sprawdzone na klastrze i trzeba to zweryfikować w kroku 9.

## Pliki

| Plik | Do czego |
|---|---|
| `bootstrap/application-controller.yaml` | controller Argo CD v3.6.3 na hubie (SA, RBAC, StatefulSet, NetworkPolicy) |
| `bootstrap/argocd-cm-health-check.yaml` | health check dla `argoproj.io/Application` (jeden klucz w `argocd-cm`) |
| `bootstrap/principal-networkpolicy-spoke.yaml` | agent -> principal (już zastosowany, patrz krok 1) |
| `bootstrap/argocd-notifications-cm.yaml` | webhook do analyzera (opcjonalny dla samego testu kolejności) |
| `app-of-apps-agent/` | chart parenta, dzieci z etykietą `argocd-agent=true` i `sync-wave` |
| `pipeline/10-tasks.yaml` | Task `argocd-parent` (nowy) |
| `pipeline/20-pipeline.yaml` | Pipeline `tenant0-parent-upgrade` (nowy) |
| `pipeline/31-pipelinerun-parent.yaml` | wyzwalacz |

## Kroki (wykonujesz je ty)

Wszystkie polecenia z katalogu `~/code/tenant0-k3s`, z jawnym kontekstem `k3s`.

### 0. Git: tag z nowym chartem i app22
Parent i dzieci czytają repo z gita (spoke też), więc tag musi zawierać `app-of-apps-agent/`, `apps/app22/`.
```bash
cd ~/code/tenant0-k3s
git add app-of-apps-agent apps/app22 bootstrap pipeline DEPLOY-app-of-apps-agent.md
git commit -m "v37 app22 + app-of-apps-agent (sync-wave on hub)"
git tag v37
git push origin main v37
```

### 1. Sprzątanie po poprzednich próbach
Stary `app21` (utworzony przez pipeline, bez etykiety agenta) kolidowałby z dzieckiem parenta.
```bash
kubectl --context k3s -n argocd delete application app21 --ignore-not-found
kubectl --context k3s -n argocd delete application guestbook --ignore-not-found
kubectl --kubeconfig ~/.kube/spoke1-tenant0.kubeconfig delete namespace tenant0-app21 --ignore-not-found
kubectl --kubeconfig ~/.kube/spoke1-tenant0.kubeconfig -n argocd get applications     # ma nie zwrócić app21/guestbook
```
Port-forward na spoke (`kubectl --context k3s -n spoke1-tenant0 port-forward svc/spoke1-tenant0 10961:443`) musi działać.

### 2. NetworkPolicy agent -> principal (jeśli jeszcze nie zastosowana)
```bash
kubectl --context k3s -n argocd get networkpolicy argocd-agent-principal-from-spoke \
  || kubectl --context k3s apply -f bootstrap/principal-networkpolicy-spoke.yaml
```

### 3. Application-controller na hubie
```bash
kubectl --context k3s apply -f bootstrap/application-controller.yaml
kubectl --context k3s -n argocd rollout status statefulset/argocd-application-controller --timeout=180s
kubectl --context k3s -n argocd logs statefulset/argocd-application-controller --tail=10
```
Oczekiwane: pod `Running`, w logu brak `forbidden`. Pamiętaj: sekret klastra agenta ma `skip-reconcile: "true"`
(sprawdzone), więc controller nie dotknie dzieci kierowanych do `spoke1-tenant0`.

### 4. Health check Application
```bash
kubectl --context k3s apply --server-side --field-manager=tenant0 -f bootstrap/argocd-cm-health-check.yaml
kubectl --context k3s -n argocd rollout restart statefulset/argocd-application-controller
kubectl --context k3s -n argocd rollout status statefulset/argocd-application-controller --timeout=180s
kubectl --context k3s -n argocd get cm argocd-cm -o jsonpath='{.data.resource\.customizations\.health\.argoproj\.io_Application}' | head -3
```

### 5. Principal ma widzieć tylko aplikacje z etykietą `argocd-agent=true`
Bez tego principal próbuje routować parenta (destination = hub, brak nazwy agenta) i loguje błąd.
Najpierw oznacz AppProject, bo selektor dotyczy też AppProjectów (kod principala filtruje ich listę i watch tym samym selektorem). **[HIPOTEZA]** Bez etykiety `default` przestałby być wysyłany na spoke.
```bash
kubectl --context k3s -n argocd label appproject default argocd-agent=true --overwrite
kubectl --context k3s -n argocd patch cm argocd-agent-params --type merge \
  -p '{"data":{"principal.label-selector":"argocd-agent=true"}}'
kubectl --context k3s -n argocd rollout restart deployment/argocd-agent-principal
kubectl --context k3s -n argocd rollout status deployment/argocd-agent-principal --timeout=120s
kubectl --context k3s -n spoke1-tenant0 logs -l app.kubernetes.io/name=argocd-agent-agent --tail=5
```
Oczekiwane: agent łączy się ponownie (brak `connection refused`). `agent.label-selector` na spoke zostawiam puste
(jest tylko zabezpieczeniem dla spoke'ów z cudzymi aplikacjami, tu ich nie ma).

### 6. Notyfikacje do analyzera (opcjonalne dla testu kolejności)
```bash
kubectl --context k3s apply --server-side --force-conflicts -f bootstrap/argocd-notifications-cm.yaml
kubectl --context k3s -n argocd rollout restart deploy/argocd-notifications-controller
```

### 7. Task, Pipeline
```bash
kubectl --context k3s apply -f pipeline/10-tasks.yaml -f pipeline/20-pipeline.yaml
```

### 8. Uruchomienie
Terminal 1 (obserwacja kolejności):
```bash
kubectl --context k3s -n argocd get applications -w
```
Terminal 2:
```bash
kubectl --context k3s create -f pipeline/31-pipelinerun-parent.yaml
kubectl --context k3s -n tenant0-pipeline get pipelinerun -w
```

### 9. Co sprawdzić (to są hipotezy, które test potwierdzi lub obali)
1. `app22` powstaje **dopiero po** tym, jak `app21` jest `Healthy` (PreSync, Sync i PostSync zakończone).
```bash
kubectl --context k3s -n argocd get applications -o custom-columns='NAME:.metadata.name,WAVE:.metadata.annotations.argocd\.argoproj\.io/sync-wave,CREATED:.metadata.creationTimestamp,SYNC:.status.sync.status,HEALTH:.status.health.status'
kubectl --context k3s -n argocd get application tenant0-spoke1-tenant0 -o jsonpath='{.status.operationState.phase}{"\n"}{.status.operationState.message}{"\n"}'
```
2. **[HIPOTEZA]** Parent poprawnie czyta zdrowie dzieci, mimo że ich status zapisuje principal (a nie hubowy controller).
   Jeśli parent po fali 0 wisi w `Progressing`, a `app21` jest `Healthy`, to hipoteza jest fałszywa. Wtedy zbieram `kubectl ... describe application` parenta i logi controllera.
3. **[HIPOTEZA]** Parent nie walczy z principalem o te same obiekty dzieci (spec vs status). Objaw: kolejne `Sync` parenta bez zmiany revision.
4. Dzieci faktycznie są obsługiwane na spoke:
```bash
kubectl --kubeconfig ~/.kube/spoke1-tenant0.kubeconfig -n argocd get applications
kubectl --kubeconfig ~/.kube/spoke1-tenant0.kubeconfig get pods -A | grep -E "app21|app22"
```
5. Logi controllera hubu pokazują tylko parenta, a nie dzieci:
```bash
kubectl --context k3s -n argocd logs statefulset/argocd-application-controller --tail=200 | grep -E "app21|app22|tenant0-spoke1" | cut -c1-200
```

### 10. Kolejny upgrade
Zmień pliki w `apps/`, wypchnij **nowy** tag `v38`, ustaw go w `target-revision` w `pipeline/31-pipelinerun-parent.yaml`
i utwórz PipelineRun jeszcze raz (krok 8). Kolejność aplikacji zmieniasz w `app-of-apps-agent/values.yaml` (pole `wave`).
Uwaga: jeśli po zmianie tagu wyrenderowane manifesty dzieci są identyczne, Argo CD nie wystartuje operacji i `argocd-wait`
skończy się timeoutem (znane ograniczenie, patrz `HANDOFF.md` sekcja 6).

## Wycofanie zmian (jeśli coś pójdzie źle)
```bash
kubectl --context k3s -n argocd delete application tenant0-spoke1-tenant0 app21 app22 --ignore-not-found
kubectl --context k3s -n argocd patch cm argocd-agent-params --type merge -p '{"data":{"principal.label-selector":""}}'
kubectl --context k3s -n argocd rollout restart deployment/argocd-agent-principal
kubectl --context k3s delete -f bootstrap/application-controller.yaml --ignore-not-found
```
Po usunięciu parenta z `prune: true` Argo CD usuwa jego dzieci. Dzieci nie mają finalizera, więc zasoby workloadu
na spoke zostają. Usuń je ręcznie przez `kubectl --kubeconfig ... delete namespace tenant0-app21 tenant0-app22`.

## Czego ten wariant nie rozwiązuje
- ApplicationSet na hubie nadal nie działa (brak `applicationset-controller`).
- Test PKI i `principal.listen.host: 0.0.0.0` są tylko testowe (jak wcześniej).
- Obraz controllera `quay.io/argoproj/argocd:v3.6.3` jest ten sam co `argocd-server` na hubie. Przy zmianie wersji trzeba je podnosić razem.
