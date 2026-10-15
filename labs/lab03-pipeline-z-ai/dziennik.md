# Dziennik — lab03 (pipeline z AI)

| # | Objaw (krok, komunikat) | Hipoteza (AI / ja) | Jak sprawdziłem | Poprawka | Status |
|---|---|---|---|---|---|
| 1 | `gh variable set AWS_ROLE_ARN` → `HTTP 403: You must have repository write permissions...` (URL błędu wskazywał na `dawid-marniok/aiops-praktyka`) | AI: `gh` dla forka domyślnie rozwiązuje repo do parenta (upstream), nie do forka, mimo `gh repo set-default` | Odczytałem URL w komunikacie błędu — prowadził do upstreamu, nie do `DKUMK/aiops-praktyka` | `export GH_REPO=DKUMK/aiops-praktyka` przed `gh variable set` | ✅ |
| 2 | `AWS_ROLE_ARN`/`ECR_REGISTRY` — niejasne czy to sekrety | Ja: podejrzewałem, że trzeba użyć `gh secret set` | Sprawdziłem README: "to nie są sekrety", potwierdzone też wymogiem checklisty "Zmienne z vars, nie z secrets" | Użyto `gh variable set` (GitHub Variables), nie `gh secret set` | ✅ |

## Sugestie AI, których nie wdrożyłem

1. **Cache warstw Dockera w etapie build+push.** AI proponowało job typu matrix (4 serwisy) z `docker/build-push-action` i `cache-from/cache-to: type=gha`, co dałoby realne cache'owanie warstw między przebiegami CI. Nie wdrożyłem tego — wybrałem wywołanie istniejących celów `app/Makefile` (`make ecr-login build push`) 1:1, żeby pipeline budował obrazy dokładnie tą samą komendą `docker build`, którą uczestnik odtwarza lokalnie, bez duplikowania logiki buildu w YAML (ryzyko rozjechania się z Makefile przy przyszłej zmianie). Koszt: każdy push na `main` buduje 4 obrazy od zera, bez cache'u warstw.

2. **Smoke test `/readyz` zamiast `/healthz`.** `orders-api` ma dwa endpointy: `/healthz` (trywialny, zawsze `{"status":"ok"}`, bez sprawdzania zależności) i `/readyz` (realnie odpytuje Postgresa przez `db.ping()`). AI zwróciło uwagę, że `/readyz` byłby bardziej wartościowym smoke testem po wdrożeniu — wykryłby np. problem z połączeniem do bazy, czego `/healthz` nigdy nie zgłosi. Zostałem przy `/healthz`, bo tak literalnie brzmi wymaganie w README laba ("smoke test `/healthz` z wnętrza klastra"); gdybym to zmieniał samodzielnie, odszedłbym od zadanej specyfikacji.
