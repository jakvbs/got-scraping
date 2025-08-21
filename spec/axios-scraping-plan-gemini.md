Jasne, oto szczegółowy plan, jak można rozszerzyć bibliotekę `axios`, aby naśladowała kluczowe funkcjonalności `got-scraping`. Plan opiera się na analizie dostarczonego kodu `got-scraping` i tłumaczy jego mechanizmy na ekosystem `axios`.

### **Cel: Stworzenie `axios-scraping`**

Naszym celem jest stworzenie wrappera (opakowania) na `axios`, który będzie dostarczał te same zaawansowane funkcje co `got-scraping`, takie jak:
1.  Automatyczne generowanie nagłówków przeglądarki.
2.  Zaawansowana obsługa proxy (HTTP, HTTPS, HTTP/2).
3.  Modyfikacja "odcisku palca" TLS (TLS fingerprint).
4.  Wsparcie dla HTTP/2.
5.  Większa odporność na błędy serwerów (np. niepoprawna kompresja).
6.  Utrzymywanie spójności nagłówków w ramach sesji.

---

### **1. Filozofia i Architektura Rozwiązania**

`got` opiera swoje rozszerzenia na systemie **haków (hooks)**. Odpowiednikiem tego mechanizmu w `axios` są **interceptory (interceptors)**. Będą one fundamentem naszej implementacji.

Kluczowe, niskopoziomowe operacje (obsługa proxy, TLS, HTTP/2) w `got-scraping` są realizowane przez niestandardowe **agenty HTTP/HTTPS**. `axios` w środowisku Node.js również pozwala na podpięcie własnych agentów poprzez opcje konfiguracyjne `httpAgent` i `https.Agent`.

**Główna strategia:**
Stworzymy funkcję fabryki, np. `createAxiosScraping(defaultOptions)`, która zwróci nową, skonfigurowaną instancję `axios`. Ta instancja będzie miała wpięte interceptory odpowiedzialne za implementację logiki `got-scraping`.

---

### **2. Kluczowe Funkcjonalności do Zaimplementowania (Krok po Kroku)**

#### **2.1. Generator Nagłówków Przeglądarki**

`got-scraping` używa biblioteki `header-generator` do tworzenia realistycznych nagłówków.

*   **Zależność:** `header-generator`
*   **Implementacja:** Stworzymy **interceptor żądania (request interceptor)**.

**Logika interceptora:**
1.  Sprawdzi, czy w konfiguracji żądania `axios` znajduje się opcja `useHeaderGenerator: true` (domyślnie włączona).
2.  Jeśli tak, zainicjalizuje `HeaderGenerator` z opcjami podanymi w `headerGeneratorOptions`.
3.  Wygeneruje nagłówki, biorąc pod uwagę wersję protokołu HTTP (więcej o tym w punkcie 2.4).
4.  Połączy wygenerowane nagłówki z nagłówkami podanymi przez użytkownika, dając pierwszeństwo tym drugim.
5.  `got-scraping` dba również o kolejność i wielkość liter w nagłówkach (PascalCase) za pomocą `TransformHeadersAgent`. W `axios` nie mamy bezpośredniej kontroli nad tym, jak Node.js wysyła nagłówki, ale możemy to osiągnąć, tworząc własnego, prostego agenta opakowującego (wrapper agent).

*   **Plik:** `src/interceptors/browser-headers.ts`

#### **2.2. Zaawansowana Obsługa Proxy**

Wbudowana obsługa proxy w `axios` jest podstawowa. `got-scraping` dynamicznie wybiera odpowiedniego agenta proxy w zależności od protokołu proxy i serwera docelowego.

*   **Zależności:**
    *   `hpagent` (dla proxy HTTP/HTTPS w Node.js, dobra alternatywa dla starych bibliotek).
    *   `http2-wrapper` (do obsługi proxy HTTP/2, tak jak w `got-scraping`).
*   **Implementacja:** Logika zostanie umieszczona w **interceptorze żądania**, który uruchomi się przed interceptorem nagłówków.

**Logika interceptora:**
1.  Sprawdzi, czy w konfiguracji żądania podano `proxyUrl`.
2.  Jeśli tak, sparsuje URL proxy, aby uzyskać protokół, host, port i dane uwierzytelniające.
3.  Na podstawie protokołu proxy (`http:`, `https:`) i protokołu docelowego (`http:`, `https:`) zdecyduje, jakiego agenta użyć.
    *   **Cel `https://` przez proxy `http://`:** Użyje agenta, który potrafi obsłużyć tunelowanie `CONNECT` (np. z `hpagent`).
    *   **Cel `https://` przez proxy `https://` (z obsługą HTTP/2):** Sprawdzi, czy proxy wspiera HTTP/2 (ALPN). Jeśli tak, użyje agenta z `http2-wrapper` (np. `Http2OverHttp2`, `HttpsOverHttp2`). W przeciwnym razie użyje standardowego agenta tunelującego.
4.  Stworzy instancję odpowiedniego agenta z przekazanymi opcjami (np. danymi do logowania).
5.  Przypisze stworzonego agenta do `config.httpAgent` lub `config.httpsAgent`.

*   **Plik:** `src/interceptors/proxy.ts`, `src/agents/` (katalog z implementacjami agentów).

#### **2.3. Modyfikacja Odcisku Palca TLS (TLS Fingerprint)**

`got-scraping` modyfikuje parametry połączenia TLS, aby bardziej przypominać przeglądarkę. Opcje te (`ciphers`, `sigalgs`, `ecdhCurve`, etc.) są przekazywane do agenta HTTPS.

*   **Implementacja:** Ta logika będzie częścią **interceptora proxy** (lub osobnego interceptora TLS), ponieważ to on tworzy agenta.

**Logika interceptora/agenta:**
1.  Podczas tworzenia instancji `https.Agent` (lub agenta z `hpagent`/`http2-wrapper`), przekaże do jego konfiguracji opcje TLS.
2.  Opcje te mogą być pobrane z predefiniowanego zestawu (np. `chrome`, `firefox`), podobnie jak w pliku `src/hooks/tls.ts`. Można je uzależnić od `User-Agent` wygenerowanego w pierwszym kroku.
3.  Przypisze te opcje do `config.httpsAgent`.

*   **Plik:** `src/interceptors/tls.ts` (lub zintegrowane w `proxy.ts`).

#### **2.4. Wsparcie dla HTTP/2**

`axios` domyślnie nie obsługuje HTTP/2 w Node.js. `got-scraping` włącza je domyślnie i negocjuje protokół (ALPN).

*   **Zależność:** `http2-wrapper`
*   **Implementacja:** Ponownie, kluczową rolę odegra **interceptor proxy/agenta**.

**Logika interceptora:**
1.  Domyślnie `axios-scraping` będzie próbował używać HTTP/2.
2.  Przed wysłaniem żądania, interceptor (lub agent) musi przeprowadzić negocjację ALPN z serwerem docelowym (lub przez proxy), aby dowiedzieć się, czy wspiera on `h2`.
3.  Jeśli tak, żądanie zostanie wysłane przez agenta obsługującego HTTP/2 (z `http2-wrapper`).
4.  Jeśli nie, nastąpi powrót do agenta HTTP/1.1.
5.  Ta informacja zwrotna jest ważna dla interceptora nagłówków, który musi wygenerować nagłówki odpowiednie dla danej wersji HTTP.

#### **2.5. Odporność na Błędy i Niestandardowe Opcje**

*   **`insecureHTTPParser`:** `got-scraping` domyślnie włącza tę opcję Node.js, aby akceptować niepoprawnie sformatowane odpowiedzi HTTP. Możemy to osiągnąć, przekazując tę opcję podczas tworzenia naszego niestandardowego `http.Agent`.
*   **`fixDecompress`:** `got-scraping` naprawia niepoprawnie skompresowane odpowiedzi. W `axios` możemy to zaimplementować za pomocą **interceptora odpowiedzi (response interceptor)**, który analizuje strumień odpowiedzi i w razie błędu dekompresji próbuje go naprawić. Jest to zaawansowana funkcja, ale możliwa do zrealizowania.

#### **2.6. Zarządzanie Sesją (`sessionToken`)**

`got-scraping` używa `sessionToken` do zapewnienia, że dla danego tokenu generowane są zawsze te same nagłówki.

*   **Implementacja:** Stworzymy globalny (w zasięgu modułu) magazyn, np. `WeakMap`, do przechowywania danych sesji (głównie wygenerowanych nagłówków).

**Logika w interceptorze nagłówków:**
1.  Sprawdzi, czy w konfiguracji żądania przekazano `sessionToken`.
2.  Jeśli tak, sprawdzi w `WeakMap`, czy dla tego tokenu istnieją już zapisane nagłówki.
3.  Jeśli tak, użyje ich.
4.  Jeśli nie, wygeneruje nowe nagłówki, zapisze je w `WeakMap` pod kluczem `sessionToken` i użyje ich w żądaniu.

---

### **3. Struktura Projektu i Implementacja (Pseudo-kod)**

Sugerowana struktura plików:

```
src/
├── agents/              # Implementacje niestandardowych agentów
│   ├── h1-proxy-agent.ts
│   └── transform-headers-agent.ts
├── interceptors/        # Logika interceptorów
│   ├── browser-headers.ts
│   ├── proxy.ts
│   └── tls.ts
├── types.ts             # Definicje typów
└── index.ts             # Główna funkcja fabryki `createAxiosScraping`
```

**Przykład pliku `index.ts`:**

```typescript
import axios, { AxiosInstance, AxiosRequestConfig } from 'axios';
import { applyBrowserHeadersInterceptor } from './interceptors/browser-headers';
import { applyProxyInterceptor } from './interceptors/proxy';
// ... inne importy

// Definicja niestandardowych opcji
export interface AxiosScrapingConfig extends AxiosRequestConfig {
    proxyUrl?: string;
    useHeaderGenerator?: boolean;
    headerGeneratorOptions?: any; // Typy z header-generator
    // ... inne opcje
}

export function createAxiosScraping(defaultConfig: AxiosScrapingConfig = {}): AxiosInstance {
    const instance = axios.create(defaultConfig);

    // Wpinamy interceptory w odpowiedniej kolejności
    applyProxyInterceptor(instance);
    applyBrowserHeadersInterceptor(instance);
    // ... inne interceptory

    return instance;
}

export const axiosScraping = createAxiosScraping();
```

---

### **4. Podsumowanie i Dalsze Kroki**

1.  **Rdzeń Logiki:** Przeniesienie logiki z haków `got` do interceptorów `axios`.
2.  **Niski Poziom:** Implementacja kluczowych funkcji (proxy, TLS, HTTP/2) za pomocą niestandardowych agentów HTTP/HTTPS i przypisywanie ich do konfiguracji żądania wewnątrz interceptorów.
3.  **Zależności:** Wykorzystanie istniejących bibliotek, takich jak `header-generator`, `hpagent` i `http2-wrapper`, aby nie pisać wszystkiego od zera.
4.  **API:** Stworzenie prostej w użyciu funkcji fabryki, która dostarczy w pełni skonfigurowaną instancję `axios`, zachowując jednocześnie pełną kompatybilność z jego API.

Ten plan stanowi solidną podstawę do budowy `axios-scraping`. Największym wyzwaniem będzie prawidłowe zarządzanie i komponowanie agentów w interceptorze proxy, ponieważ to tam krzyżuje się większość zaawansowanych funkcjonalności.

Na koniec warto wspomnieć, że sam `got-scraping` jest już przestarzały (EOL - End of Life), a jego twórcy rekomendują użycie `impit`. Jednakże, stworzenie podobnego narzędzia dla `axios` jest świetnym ćwiczeniem inżynierskim i może być bardzo przydatne dla osób mocno osadzonych w ekosystemie `axios`.