# Dziennik incydentu — lab07 (capstone)

| Godzina | Hipoteza (AI / ja) | Komenda / zapytanie | Wynik | Status |
|---|---|---|---|---|
| 14:27 | — (wykrycie, brak hipotezy jeszcze) | `setup/check.sh` | Check nie przechodzi od ok. 10 minut | 🔴 w toku |
| 14:37 | — (komunikacja, nie diagnoza) | Agent wg `docs/prompts/komunikat-incydentu.md` → komunikat dla zespołu developerskiego | Komunikat nr 1 wysłany (treść poniżej) | ✅ wysłano |
| 14:41 | AI (v1): nowa wersja ma pody w CrashLoopBackOff / nie przechodzą readiness | `kubectl -n dawid-k get pods -l app.kubernetes.io/name=orders-api -L app.kubernetes.io/version -o wide` + `describe pod` | niewykonane jeszcze — hipoteza oparta tylko na zgłoszeniu ("2 pody w nowej wersji nie działają") | 🔎 do weryfikacji |
| 14:41 | AI (v1): błąd aplikacyjny (5xx) w nowej wersji na endpointach menu/zamówień, pody żyją | `kubectl -n dawid-k logs deploy/kantyna-orders-api --since=15m \| grep -i error` + `kubectl -n dawid-k get configmap kantyna-flags -o jsonpath='{.data.flags\.json}'` | niewykonane jeszcze | 🔎 do weryfikacji |
| 14:41 | AI (v1): wspólna zależność (DB/payments/flags) źle skonfigurowana po rollout nowej wersji | `helm -n dawid-k history kantyna` + `pg_isready` (exec może być zablokowany politykami klastra) | niewykonane jeszcze | 🔎 do weryfikacji |

## Komunikat #1 — do zespołu developerskiego (14:37)

> Od około 10 minut automatyczna kontrola (check) nie przechodzi poprawnie. Problem trwa nieprzerwanie od tego momentu. Pełny zakres wpływu na działanie systemu nie jest jeszcze potwierdzony. Zespół przystępuje do diagnozy przyczyny awarii. Kolejna aktualizacja statusu za 15 minut.

## Diagnoza — hipotezy v1 (AI, 14:41, bez wykonanych komend)

Na podstawie zgłoszenia: namespace `dawid-k`, komponent kantyna, check nie przechodzi od ~10 min, 2 pody w nowej wersji nie działają, menu i zamówienie przez web nie działają. Żadna z poniższych hipotez nie została jeszcze potwierdzona danymi z klastra — to pierwsza wersja do porównania z kolejną iteracją po wykonaniu komend weryfikujących.

| Hipoteza | Dowód z danych zgłoszenia | Komenda weryfikująca (read-only) |
|---|---|---|
| 1. Nowa wersja ma pody w `CrashLoopBackOff` / nie przechodzą readiness | "2 w nowej wersji pody, nie działają" — tłumaczy jednocześnie fail check.sh i niedostępność menu/zamówień | `kubectl -n dawid-k get pods -l app.kubernetes.io/name=orders-api -L app.kubernetes.io/version -o wide`; `kubectl -n dawid-k describe pod <pod>` |
| 2. Błąd aplikacyjny (5xx) w nowej wersji na endpointach menu/zamówień, pody żyją | "menu przez web i zamówienie nie działa" — typowy objaw 5xx z runbooka `KantynaHighErrorRate`; rozstrzyga status podów z hipotezy 1 | `kubectl -n dawid-k logs deploy/kantyna-orders-api --since=15m \| grep -i error`; `kubectl -n dawid-k get configmap kantyna-flags -o jsonpath='{.data.flags\.json}'` |
| 3. Wspólna zależność (DB/payments/flags) źle skonfigurowana po rollout nowej wersji | Menu (odczyt) i zamówienie (zapis+płatność) padają razem — sugeruje wspólną zależność, nie błąd jednej ścieżki | `helm -n dawid-k history kantyna`; `kubectl -n dawid-k exec sts/kantyna-postgres -- pg_isready -h 127.0.0.1` *(exec może być zablokowany — alternatywa: metryka `db_pool_connections_in_use`)* |

**Brakujące dane** (zamiast zgadywania): faktyczna nazwa/godzina release'u (`helm history`), realny status i powód restartu podów nowej wersji (`get pods` + `describe`), czy błąd leży w `orders-api`, `web` czy `payments`.

## Weryfikacja hipotez v1 — wyniki (14:50)

| Godzina | Hipoteza | Komenda | Wynik | Status |
|---|---|---|---|---|
| 14:41 | `kubectl -n dawid-k get pods -o wide -L app.kubernetes.io/version` | — | Oba pody `orders-api` **Running, READY 0/1, RESTARTS 0** (nie CrashLoopBackOff) | ❌ hipoteza 1 (crash loop) odrzucona |
| 14:41 | `kubectl -n dawid-k describe pod kantyna-orders-api-...-ddc24` | — | `Warning Unhealthy ... Readiness probe failed: context deadline exceeded` (18 min, ~186x) — pod wystartował 25h temu, nie jest to "nowa wersja" | Potwierdza: READY=0/1, nie crash loop |
| 14:41 | `kubectl -n dawid-k logs <orders-api pod>` (oba pody) | — | `"Readiness check failed", "error": "couldn't get a connection after 30.00 sec"` co 5s w obu podach | ✅ wskazuje na wyczerpany pool połączeń do DB |
| 14:50 | `kubectl -n dawid-k get configmap kantyna-flags -o jsonpath='{.data.flags\.json}'` | — | **`db_pool_leak: true`, `slow_menu_query: true`** dla `orders-api` (reszta flag `false`) | ✅ **POTWIERDZONA PRZYCZYNA** |
| 14:50 | `helm -n dawid-k get values kantyna` | — | Computed values Helm (= `app/deploy/helm/kantyna/values.yaml` w repo): `db_pool_leak: false`, `slow_menu_query: false` | ⚠️ repo i stored Helm values są już poprawne — **drift tylko na żywym klastrze**, nie w kodzie/git |
| 14:50 | `helm -n dawid-k history kantyna` | — | Ostatni deployed release: rev 7, 15 Oct 10:46 (25h temu) — żaden release nie pokrywa się z 10-min oknem awarii | Odrzuca "nową wersję" jako przyczynę; potwierdza, że ktoś zmienił ConfigMap bezpośrednio (poza Helmem), ~18 min temu |

### Wniosek
**Przyczyna potwierdzona:** ConfigMap `kantyna-flags` został zmieniony poza Helmem (ręczny `kubectl patch`/`edit`, nie commit/release) — włączono syntetyczne flagi `db_pool_leak` i `slow_menu_query` dla `orders-api`. To wyczerpuje pool połączeń do Postgresa → `/readyz` timeoutuje → pody `orders-api` nie są Ready → `check.sh` pada, menu i zamówienia przez web nie działają. Repo (`values.yaml`) i zapisane wartości release'u Helm są poprawne (`false`) — **nie ma tu żadnej zmiany do zacommitowania w repo**, bo kod/config źródłowy już jest poprawny. To czysty drift na klastrze, poza git.

## Mitygacja — decyzja wbrew propozycji AI (14:58)

**Co proponowało AI:** uruchomić teraz lokalnie `helm -n dawid-k upgrade kantyna app/deploy/helm/kantyna --reuse-values`, żeby natychmiast odtworzyć poprawny ConfigMap z zapisanych wartości release'u.

**Co się stało:** komenda została zablokowana przez wewnętrzny klasyfikator bezpieczeństwa Claude Code (zmiana stanu klastra), bez możliwości wykonania bez dodatkowego luzowania uprawnień.

**Moja decyzja (wbrew pierwszej propozycji AI):** zamiast odblokowywać lokalne uprawnienia albo ręcznie patchować ConfigMap, wybrałem commit + push do `main` — ten sam efekt (`helm upgrade --reuse-values` w kroku deploy pipeline'u z lab03) zostaje osiągnięty przez istniejący, już zatwierdzony pipeline, bez obchodzenia lokalnych zabezpieczeń i w sposób w pełni audytowalny (log przebiegu CI). AI samo zaproponowało tę opcję jako alternatywę nr 2, dopiero po tym jak opcja 1 (bezpośrednia komenda) nie zadziałała — uznałem ją za lepsze rozwiązanie od początku do końca procesu, nie tylko jako fallback.

**Ryzyko do obserwacji:** krok `helm upgrade --reuse-values` w CI może zgłosić konflikt Server-Side Apply, jeśli pole `data.flags.json` ma teraz innego field-managera (po ręcznym `kubectl patch`). Jeśli pipeline padnie z komunikatem `conflict with "kubectl-patch"` — to nie błąd CI, trzeba będzie rozstrzygnąć pole ręcznie (bez `--force-conflicts`), zgodnie z README lab07.
