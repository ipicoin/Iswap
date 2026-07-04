# PRD — Iswap: Swap / DEX tokenów IPI

> Product Requirements Document · Fala 3 · status: **DRAFT do przeglądu**
> Repo: `ipicoin/Iswap` · Zamyka: [#1](https://github.com/ipicoin/Iswap/issues/1) · Realizuje: [#2](https://github.com/ipicoin/Iswap/issues/2)
> SSOT: base denom `nipi` (9 miejsc dziesiętnych), ticker/symbol **`IPI`** (1 IPI = 10^9 nipi), portfel `wallet-core.js`, RPC `https://ipicoin.eu/rpc`
> Druga strona pary w MVP: **testowy token cw20** (test/faucet) wdrożony na łańcuchu IPI. IBC/inne aktywa = faza 2.

---

## 1. Cel

Dostarczyć IPI DAO działającą aplikację **swap/DEX** umożliwiającą wymianę tokenów w
ekosystemie IPI — w MVP para **`nipi` (natywny, ticker IPI) ↔ testowy token cw20** — w sposób:

- **niepowierniczy** (non-custodial) — użytkownik podpisuje transakcje własnym kluczem przez `wallet-core.js`,
- **oparty o AMM** (Automated Market Maker) — brak księgi zleceń, ceny z puli płynności,
- **suwerenny** — logika wymiany działa na łańcuchu IPI (`https://ipicoin.eu/rpc`), a nie zależy od zewnętrznego operatora.

Obecne repozytorium to szkielet wygenerowany przez `create-cosmos-app` (przykład *swap*),
zakodowany na sztywno pod sieć **Osmosis** (`osmo-query`, moduły GAMM/poolmanager,
`cosmos-kit`). Ten PRD definiuje **czym Iswap ma być dla IPI** i domyka [#1](https://github.com/ipicoin/Iswap/issues/1)
(„undefined functionality").

## 2. Użytkownicy (persony)

| Persona | Potrzeba | Kluczowa historia |
| --- | --- | --- |
| **Posiadacz IPI** | Wymienić `nipi` na token pary (w MVP: testowy token cw20) bez zaufanego pośrednika | „Chcę zamienić 100 IPI na testowy token cw20 po znanej cenie" |
| **Dostawca płynności (LP)** | Zarabiać na opłatach dostarczając kapitał do puli | „Chcę wpłacić parę i otrzymać udziały LP" (faza 2) |
| **DAO / skarbnik** | Zarządzać parametrami rynku (opłaty, listing par) przez governance | „Chcę utworzyć nową pulę i ustawić opłatę" |
| **Deweloper / integrator** | Programowo kwotować i wykonywać swap | „Chcę wywołać kontrakt/RPC z zewnętrznej apki" |

## 3. Zakres MVP

**W zakresie (MVP):**

Para MVP = **`nipi` (natywny, ticker IPI) ↔ testowy token cw20**. Ponieważ IPI ma
obecnie tylko token natywny `nipi` (IBC odroczone do fazy 2), drugą stroną puli jest
**prosty kontrakt cw20** (test/faucet token) wdrożony na łańcuchu IPI — dostarcza realny,
sparowalny asset do uruchomienia i testów AMM end-to-end bez zależności od IBC.

1. Połączenie portfela przez `wallet-core.js` (adres, saldo, podpis).
2. **Wdrożenie testowego tokena cw20** — standardowy kontrakt `cw20-base` (mint/faucet
   do testów), zarejestrowany w konfiguracji jako drugi asset pary.
3. Utworzenie puli `nipi ↔ cw20-test` na kontrakcie AMM (bootstrap rezerw z faucetu).
4. Wybór pary tokenów (from → to) z listy wspieranych aktywów (natywny `nipi` + cw20).
5. **Kwotowanie** — wyliczenie kwoty wyjściowej z rezerw puli (`x*y=k`), z opłatą puli.
6. **Slippage tolerance** — presety (0.5% / 1% / 3%) + wartość własna; wyliczenie `minAmountOut`.
7. **Swap** — złożenie i podpisanie transakcji, potwierdzenie, obsługa błędów.
8. Wskaźniki: cena, price impact, opłata, minimalny odbiór, trasa (route).

**Poza MVP (faza 2+):** patrz §11.

## 4. Model DEX — rekomendacja

### Decyzja: **natywny AMM (`x*y=k`) na CosmWasm na łańcuchu IPI**, z IBC/Osmosis jako faza 2.

### Dwie rozważane opcje

**Opcja A — natywny AMM na CosmWasm (rekomendowana):**
kontrakt puli constant-product (`x*y=k`) wdrożony na łańcuchu IPI (moduł `wasmd`).
Zamiast pisać od zera — **fork audytowanego kontraktu** (Astroport pair-xyk / WYND DEX)
i dostosowanie do `nipi` + testowego cw20. (GAMM to moduł Go w Osmosis — **nie istnieje**
jego audytowany port CosmWasm, więc nie jest opcją forka.)

**Opcja B — integracja z istniejącym DEX (Osmosis via IBC):**
token IPI transferowany po IBC na Osmosis, pula GAMM na Osmosis, front kwotuje/wykonuje
przez `osmo-query` (jak obecny szkielet).

### Macierz decyzyjna

| Kryterium | A: CosmWasm na IPI | B: Osmosis/IBC |
| --- | --- | --- |
| Suwerenność DAO | ✅ pełna kontrola pul, opłat, listingu | ❌ zależność od zewnętrznej sieci i jej governance |
| Płynność startowa | ⚠️ trzeba zbootstrapować | ⚠️ trzeba dostarczyć na Osmosis + relayer IBC |
| Dostęp do rynku zewn. | ❌ początkowo izolowany | ✅ ekspozycja na cały Interchain |
| Ryzyko techniczne | ⚠️ audyt kontraktu, wasmd musi być włączony | ⚠️ zależność od relayerów IBC, kanałów, denomów IBC |
| Zgodność ze szkieletem | ⚠️ trzeba wymienić warstwę Osmosis | ✅ szkielet już pod Osmosis |
| Koszt utrzymania | średni (własny kontrakt) | średni (monitoring IBC/relayer) |

### Uzasadnienie

IPI to **suwerenny łańcuch** z własnym tokenem (`nipi`, 9 dec, RPC `ipicoin.eu/rpc`).
Wartość dla DAO leży w kontroli nad rynkiem własnego tokena (opłaty jako przychód DAO,
governance nad listingiem, brak zależności od cudzej sieci). Model B uzależnia rdzenną
funkcję (swap IPI) od zewnętrznej sieci, mostów IBC i relayerów — to ryzyko operacyjne
i strategiczne nieproporcjonalne do MVP. Model A daje działający swap natychmiast na
własnym łańcuchu.

Aby ograniczyć ryzyko bezpieczeństwa, **nie piszemy AMM od zera** — forkujemy audytowany
kontrakt (Astroport pair-xyk / WYND) i dostosowujemy denominacje i UI.
Model B nie jest odrzucony — wchodzi jako **faza 2** (listing na Osmosis po IBC) dla
dostępu do zewnętrznej płynności i widoczności, gdy rynek natywny okrzepnie.

> Konsekwencja dla kodu: obecna warstwa `osmo-query`/GAMM (`hooks/usePools.ts`,
> `hooks/useSwap.tsx`, `hooks/useQueryHooks.ts`) zostanie zastąpiona klientem CosmWasm
> (`@cosmjs/cosmwasm-stargate`, już w zależnościach) kwerendującym kontrakt puli IPI.

## 5. Funkcje

### 5.1 Wybór pary
- Lista wspieranych tokenów z konfiguracji (`config/`), rozszerzonej o `chainconfig` IPI:
  natywny `nipi` (ticker IPI) + **testowy token cw20** (adres kontraktu w konfiguracji).
- Domyślnie para `nipi` (IPI) ↔ **testowy token cw20**. Kolejne aktywa (IBC) = faza 2.
- Zamiana kierunku (↑↓) from/to.

### 5.2 Kwotowanie (quote)
- Pobranie rezerw puli z kontraktu (query `Pool` / `Simulation`).
- Wyliczenie `amountOut = (reserveOut * amountIn * (1 - fee)) / (reserveIn + amountIn * (1 - fee))`.
- Prezentacja: cena, price impact, opłata puli, trasa (route) — dla par bez bezpośredniej puli routing wieloetapowy (faza 2).

### 5.3 Slippage
- Presety **0.5% / 1% / 3%** + pole własne (walidacja 0–50%).
- `minAmountOut = quote * (1 - slippage)`; przekazany do msg swap jako zabezpieczenie.
- Ostrzeżenie przy price impact > próg (np. 5%).

### 5.4 Swap
- Budowa `MsgExecuteContract` (swap) → podpis przez `wallet-core.js` → broadcast na `ipicoin.eu/rpc`.
- Stany UI: idle → quoting → awaiting-signature → broadcasting → success/error.
- Toast z linkiem do eksploratora; odświeżenie sald.

### 5.5 (Faza 2) Płynność
- Add liquidity (deposit pary → mint LP shares).
- Remove liquidity (burn LP shares → withdraw).
- Widok pul: TVL, APR z opłat, udział użytkownika.

## 6. Integracje

| Element | Rola | Uwagi |
| --- | --- | --- |
| **Kontrakt AMM (CosmWasm)** | logika AMM (pool `x*y=k`, swap, LP) | fork audytowanego AMM (Astroport pair-xyk / WYND); scaffolding przez `cw-template`; kodegen TS przez `@cosmwasm/ts-codegen` |
| **Testowy token cw20 (MVP)** | druga strona pary `nipi ↔ cw20-test` | standardowy `cw20-base` (mint/faucet do testów); swap z cw20 przez `Cw20::Send` z hookiem do kontraktu AMM, swap z natywnego `nipi` przez `funds`; adres w `chainconfig` |
| **`wallet-core.js`** | połączenie portfela, adres, saldo, podpis | zastępuje `cosmos-kit`/keplr ze szkieletu; interfejs: `getAddress()`, `getBalance()`, `signAndBroadcast()` |
| **RPC** | zapytania łańcucha + broadcast | `https://ipicoin.eu/rpc` (`@cosmjs/cosmwasm-stargate` `SigningCosmWasmClient`) |
| **`chainconfig`** | metadane łańcucha IPI | chain-id, `bech32Prefix`, denom `nipi`, `coinDecimals: 9`, endpoint RPC/REST; zastępuje wpis `chain-registry`/`osmosis` |
| **IBC** | (faza 2) transfer aktywów, listing na Osmosis | kanały IBC, denomy `ibc/...`, relayer |

### Model danych (SSOT)
```
Denom:   base = "nipi", exponent = 9, symbol/ticker = "IPI"  →  1 IPI = 1_000_000_000 nipi
AssetInfo (Astroport-style):
         native → { native: "nipi" }
         cw20   → { token: { contract_addr: "<adres cw20-test>" } }
Coin:    { denom: "nipi", amount: "<u128 w jednostkach bazowych>" }   // asset natywny
Cw20Amount: { contract_addr, amount: "<u128>" }                        // testowy token cw20
Token (UI): { symbol, kind: "native"|"cw20", denom|contractAddr, decimals, logo, amount(display), priceUsd? }
Pool:    { id, assets: [AssetInfo × reserve], totalShares, swapFee }   // MVP: [ {native:nipi}, {cw20:test} ]
Quote:   { amountIn, amountOut, price, priceImpact, fee, minAmountOut, route[] }
```

## 7. User stories + kryteria akceptacji

**US-1 — Połączenie portfela**
Jako posiadacz IPI chcę połączyć portfel, aby zobaczyć saldo.
- ✅ Klik „Connect" wywołuje `wallet-core.js`; po sukcesie widoczny skrócony adres.
- ✅ Widoczne saldo `nipi` w jednostkach display (÷10^9).
- ✅ Błąd/odrzucenie połączenia pokazuje czytelny komunikat, brak crasha.

**US-1b — Testowy token cw20 (MVP)**
Jako tester chcę zdobyć testowy token cw20, aby móc wykonać swap w parze `nipi ↔ cw20`.
- ✅ Kontrakt `cw20-base` wdrożony na łańcuchu IPI; adres w `chainconfig`.
- ✅ Funkcja mint/faucet pozwala testerowi uzyskać saldo testowego cw20.
- ✅ Saldo cw20 (query `Balance`) widoczne w UI obok salda `nipi`.

**US-2 — Kwotowanie swapu**
Jako użytkownik chcę wpisać kwotę i zobaczyć, ile dostanę.
- ✅ Po wpisaniu `amountIn` (np. `nipi` → testowy cw20) w ≤1 s pojawia się `amountOut`, cena, price impact, opłata.
- ✅ Zmiana pary/kierunku (`nipi ↔ cw20`) przelicza kwotowanie.
- ✅ Brak puli dla pary → jasny komunikat „brak płynności", przycisk swap zablokowany.

**US-3 — Ustawienie slippage**
Jako użytkownik chcę ustawić tolerancję poślizgu.
- ✅ Presety 0.5/1/3% + własna wartość; walidacja zakresu.
- ✅ `minAmountOut` widoczny i zmienia się z tolerancją.
- ✅ Ostrzeżenie przy price impact powyżej progu.

**US-4 — Wykonanie swapu**
Jako użytkownik chcę wykonać wymianę.
- ✅ „Swap" buduje msg (natywny `nipi` → `MsgExecuteContract` z `funds`; testowy cw20 → `Cw20::Send` z hookiem swap), prosi o podpis w `wallet-core.js`, broadcastuje na `ipicoin.eu/rpc`.
- ✅ Sukces: toast + tx hash (link do eksploratora), salda odświeżone.
- ✅ Transakcja zwrotna, gdy odbiór < `minAmountOut` (ochrona przed poślizgiem).
- ✅ Błąd (brak gazu/odrzucenie/timeout) → komunikat, stan wraca do edytowalnego.

**US-5 (faza 2) — Dostarczenie płynności**
Jako LP chcę wpłacić parę i otrzymać udziały.
- ✅ Deposit mintuje LP shares proporcjonalnie do rezerw; withdraw je pali.

## 8. Model architektury (front)

```
pages/index.tsx
 └─ components/swap/Swap.tsx        (orkiestracja UI)
     ├─ SwapFromTo / SwapTokenInput (wybór pary, kwota)
     ├─ SwapSlippage                (tolerancja)
     ├─ SwapDetails / SwapPrice     (cena, impact, fee, minOut)
     └─ SwapButton                  (akcja)
 hooks/
  ├─ useSwap        → orkiestracja: quote + build msg + broadcast   (przepisany z osmo-query na CosmWasm)
  ├─ usePools       → query rezerw z kontraktu CosmWasm             (zamiast gamm.usePools)
  ├─ useBalances    → salda z RPC IPI
  ├─ usePrices      → wycena (oracle/pula; faza 2 zewn. cena)
  └─ useTx          → podpis+broadcast przez wallet-core.js         (zamiast cosmos-kit)
 config/
  └─ chainconfig IPI (chain-id, prefix, denom nipi/9, RPC)          (zamiast wpisu osmosis)
```

## 9. Ryzyka

| Ryzyko | Opis | Mitygacja |
| --- | --- | --- |
| **Impermanent loss** | LP tracą względem HODL przy rozjeździe cen | dokumentacja/ostrzeżenie w UI (faza 2); wybór opłaty puli |
| **MEV / front-running / sandwich** | boty wyprzedzają swap użytkownika | `minAmountOut` obowiązkowy; limit price impact; docelowo mempool ochrona / threshold enc. |
| **Płynność startowa** | pusta/płytka pula = wysoki impact | program bootstrapu płynności DAO; limit wielkości swapu vs rezerwy |
| **Bezpieczeństwo kontraktu** | bug = utrata środków | fork **audytowanego** AMM, testy, audyt przed mainnet |
| **Precyzja/zaokrąglenia** | `nipi` = 9 dec, `x*y=k` z u128 | BigNumber/u128, zaokrąglanie na niekorzyść usera, testy własności |
| **Zależność RPC** | jeden endpoint `ipicoin.eu/rpc` | fallback endpointy, retry, health-check |
| **Ryzyko IBC (faza 2)** | zawieszone kanały, denomy `ibc/...` | monitoring relayerów, whitelist kanałów |

## 10. Metryki sukcesu

- Udany swap end-to-end na testnecie IPI (podpis `wallet-core.js`, broadcast RPC).
- Kwotowanie zgodne z rzeczywistym wykonaniem w granicach tolerancji.
- Zero utraty środków przez brak `minAmountOut`.
- Czas kwotowania < 1 s; TTFB UI akceptowalny.

## 11. Poza zakresem (out of scope)

- Order book / limit orders / perpetuals.
- Agregacja płynności z wielu DEX / smart order routing cross-chain.
- Mostkowanie fiat (on/off-ramp).
- Zaawansowane typy pul (stableswap/concentrated liquidity) — kandydat na fazę 3.
- Rewards/farming/gauge — faza 3 (governance DAO).
- Listing na Osmosis via IBC — **faza 2** (świadomie po MVP natywnym).

## 12. Zamyka #1

Issue [#1](https://github.com/ipicoin/Iswap/issues/1) („undefined functionality, request for
precising what it should do") wskazywał, że repozytorium to nienazwany szkielet
`create-cosmos-app` bez zdefiniowanego celu. Ten PRD domyka #1, ustalając jednoznacznie:

- **Czym Iswap jest:** niepowierniczy AMM DEX dla tokenów IPI na łańcuchu IPI.
- **Co robi w MVP:** connect → (faucet cw20) → wybór pary `nipi ↔ cw20-test` → kwotowanie → slippage → swap (§3, §5).
- **Jak (model):** natywny CosmWasm `x*y=k` na IPI; para MVP `nipi ↔ testowy token cw20`; Osmosis/IBC jako faza 2 (§4).
- **Z czym się integruje:** `wallet-core.js`, RPC `ipicoin.eu/rpc`, `chainconfig`, base denom `nipi`/9 (ticker IPI), kontrakt `cw20-base` testowy (§6).
- **Czego NIE robi:** §11.

Dalsze wątki suwerenności/tożsamości DAO: `universal-independency-declaration#1`.

---

## Powiązania

- Issue #2 (implementacja Fala 3): ten PRD jest jej podstawą.
- Issue #1: domknięty przez §12.
- `universal-independency-declaration#1`: kontekst suwerenności IPI DAO.
