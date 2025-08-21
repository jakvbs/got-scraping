# Plan rozszerzenia Axios o funkcjonalność podobną do got-scraping

## 1. **Architektura rozwiązania**
Utworzenie biblioteki `axios-scraping` która będzie nakładką na axios (podobnie jak got-scraping jest nakładką na got):

```
axios-scraping/
├── src/
│   ├── index.ts                  # Główny punkt wejścia
│   ├── interceptors/             # Interceptory axios
│   │   ├── browser-headers.ts    # Generowanie nagłówków przeglądarki
│   │   ├── proxy.ts              # Obsługa proxy
│   │   ├── tls.ts                # Konfiguracja TLS
│   │   ├── referer.ts            # Obsługa referer
│   │   └── insecure-parser.ts    # Niestandardowy parser HTTP
│   ├── adapters/                 # Adaptery HTTP
│   │   ├── http2-adapter.ts      # Adapter HTTP/2
│   │   └── proxy-adapter.ts      # Adapter proxy
│   ├── agents/                   # Agenty HTTP
│   │   ├── transform-headers.ts  # Transformacja nagłówków
│   │   └── proxy-agents.ts       # Agenty proxy
│   ├── types.ts                  # Definicje typów
│   └── utils/                    # Narzędzia pomocnicze
│       ├── header-generator.ts   # Integracja z header-generator
│       └── protocol-resolver.ts  # Rozpoznawanie protokołów
├── package.json
└── tsconfig.json
```

## 2. **Kluczowe komponenty**

### A. **Interceptory Axios**
- **Request Interceptor**: Modyfikacja żądań przed wysłaniem
  - Generowanie nagłówków przeglądarki
  - Konfiguracja TLS
  - Ustawienie proxy
  - Obsługa sesji

- **Response Interceptor**: Przetwarzanie odpowiedzi
  - Obsługa przekierowań z referer
  - Dekompresja

### B. **Adapter HTTP/2**
Utworzenie niestandardowego adaptera axios dla HTTP/2:
```typescript
// Wykorzystanie http2-wrapper jak w got-scraping
import { auto } from 'http2-wrapper';

const http2Adapter = (config) => {
  // Negocjacja ALPN
  // Obsługa HTTP/2 lub fallback do HTTP/1.1
};
```

### C. **Generowanie nagłówków przeglądarki**
Integracja z `header-generator`:
```typescript
interface AxiosScrapingConfig extends AxiosRequestConfig {
  useHeaderGenerator?: boolean;
  headerGeneratorOptions?: HeaderGeneratorOptions;
  sessionToken?: object;
}
```

### D. **Obsługa proxy**
- Wsparcie dla HTTP/HTTPS proxy
- Automatyczna detekcja protokołu proxy
- Obsługa HTTP/2 przez proxy

## 3. **Implementacja głównej funkcji**

```typescript
import axios, { AxiosInstance } from 'axios';
import { HeaderGenerator } from 'header-generator';

export function createAxiosScraping(config?: AxiosScrapingConfig): AxiosInstance {
  const instance = axios.create({
    timeout: 60000,
    maxRedirects: 5,
    validateStatus: () => true, // Nie rzucaj błędów dla statusów HTTP
    ...config
  });

  // Dodanie interceptorów
  instance.interceptors.request.use(browserHeadersInterceptor);
  instance.interceptors.request.use(tlsConfigInterceptor);
  instance.interceptors.request.use(proxyInterceptor);
  instance.interceptors.response.use(refererInterceptor);

  // Podmiana adaptera na HTTP/2
  if (config?.http2) {
    instance.defaults.adapter = http2Adapter;
  }

  return instance;
}
```

## 4. **Kluczowe funkcjonalności do implementacji**

1. **Generowanie nagłówków**:
   - Automatyczne generowanie nagłówków pasujących do prawdziwych przeglądarek
   - Obsługa różnych wersji HTTP (1.1 vs 2.0)
   - Zachowanie spójności nagłówków w sesji

2. **Konfiguracja TLS**:
   - Dopasowanie cipher suites do przeglądarek
   - Ustawienie krzywych ECDH
   - Konfiguracja sygnatur algorytmów

3. **Obsługa proxy**:
   - Wsparcie dla uwierzytelniania
   - Automatyczna negocjacja protokołu
   - Obsługa tunelowania HTTPS

4. **HTTP/2**:
   - Automatyczna negocjacja ALPN
   - Fallback do HTTP/1.1
   - Multipleksowanie połączeń

5. **Zarządzanie sesjami**:
   - Utrzymanie stałych nagłówków w ramach sesji
   - Obsługa ciasteczek
   - Śledzenie refererów

## 5. **Różnice względem got-scraping**

**Wyzwania:**
- Axios nie ma wbudowanego wsparcia dla HTTP/2
- Brak natywnych hooków jak w got
- Inny model rozszerzalności

**Rozwiązania:**
- Wykorzystanie interceptorów zamiast hooków
- Implementacja własnego adaptera HTTP
- Użycie axios.create() do tworzenia instancji

## 6. **Kolejność implementacji**

1. **Faza 1**: Podstawowa struktura
   - Konfiguracja projektu TypeScript
   - Podstawowe typy i interfejsy
   - Szkielet głównej funkcji

2. **Faza 2**: Generowanie nagłówków
   - Integracja z header-generator
   - Interceptor nagłówków
   - Obsługa sesji

3. **Faza 3**: Konfiguracja TLS
   - Interceptor TLS
   - Dopasowanie do przeglądarek

4. **Faza 4**: Obsługa proxy
   - Podstawowe proxy HTTP/HTTPS
   - Uwierzytelnianie

5. **Faza 5**: HTTP/2
   - Adapter HTTP/2
   - Negocjacja ALPN

6. **Faza 6**: Funkcje dodatkowe
   - Obsługa refererów
   - Niestandardowy parser
   - Optymalizacje

## 7. **Przykład użycia**

```typescript
import { createAxiosScraping } from 'axios-scraping';

const axiosScraping = createAxiosScraping({
  proxyUrl: 'http://user:pass@proxy.com:8080',
  useHeaderGenerator: true,
  headerGeneratorOptions: {
    browsers: ['chrome', 'firefox'],
    devices: ['desktop'],
    operatingSystems: ['windows', 'linux']
  }
});

// Użycie jak zwykłe axios
const response = await axiosScraping.get('https://example.com');
```

## 8. **Zależności**

- `axios`: Podstawowa biblioteka
- `header-generator`: Generowanie nagłówków
- `http2-wrapper`: Obsługa HTTP/2
- `https-proxy-agent`: Agenty proxy
- `node:tls`: Konfiguracja TLS

## 9. **Analiza got-scraping**

### Kluczowe funkcjonalności z got-scraping:

1. **Hooks system** (src/index.ts:21-41):
   - `fixDecompress`: Naprawa dekompresji
   - `insecureParserHook`: Niestandardowy parser HTTP
   - `sessionDataHook`: Zarządzanie danymi sesji
   - `http2Hook`: Obsługa HTTP/2
   - `proxyHook`: Konfiguracja proxy
   - `browserHeadersHook`: Generowanie nagłówków przeglądarki
   - `tlsHook`: Konfiguracja TLS
   - `refererHook`: Obsługa referer

2. **Generowanie nagłówków** (src/hooks/browser-headers.ts):
   - Integracja z HeaderGenerator
   - Rozpoznawanie protokołu HTTP (1.1 vs 2.0)
   - Mergowanie nagłówków użytkownika z wygenerowanymi
   - Obsługa sesji dla spójności nagłówków

3. **Konfiguracja TLS** (src/hooks/tls.ts):
   - Dopasowanie ciphers do przeglądarek (Chrome, Firefox, Safari)
   - Ustawienie krzywych ECDH
   - Konfiguracja algorytmów podpisów
   - Opcje bezpieczeństwa TLS

4. **Obsługa proxy** (src/hooks/proxy.ts):
   - Wsparcie HTTP/HTTPS proxy
   - Automatyczna detekcja protokołu proxy (HTTP/2 vs HTTP/1.1)
   - Uwierzytelnianie Basic Auth
   - Różne agenty dla różnych kombinacji protokołów

5. **TransformHeadersAgent** (src/agent/transform-headers-agent.ts):
   - Transformacja nagłówków do PascalCase dla HTTP/1.1
   - Zachowanie oryginalnej wielkości liter dla nagłówków x-*

## 10. **Mapowanie funkcjonalności got-scraping na axios**

| got-scraping | axios-scraping | Implementacja |
|--------------|----------------|---------------|
| hooks.init | request interceptor | axios.interceptors.request.use() |
| hooks.beforeRequest | request interceptor | axios.interceptors.request.use() |
| hooks.beforeRedirect | response interceptor | axios.interceptors.response.use() |
| got.extend() | axios.create() | Tworzenie instancji z konfiguracją |
| TransformHeadersAgent | custom adapter | Własny adapter HTTP |
| HeaderGenerator | header interceptor | Interceptor modyfikujący nagłówki |
| HTTP/2 support | custom adapter | Adapter wykorzystujący http2-wrapper |

## 11. **Szczegóły techniczne**

### A. **Interceptor nagłówków**
```typescript
const browserHeadersInterceptor = async (config) => {
  if (config.useHeaderGenerator) {
    const headers = await generateBrowserHeaders(config);
    config.headers = { ...headers, ...config.headers };
  }
  return config;
};
```

### B. **Adapter HTTP/2**
```typescript
const http2Adapter = async (config) => {
  const { protocol } = await resolveProtocol(config.url);
  
  if (protocol === 'h2') {
    return http2Request(config);
  } else {
    return httpAdapter(config);
  }
};
```

### C. **Obsługa proxy**
```typescript
const proxyInterceptor = (config) => {
  if (config.proxyUrl) {
    const agent = createProxyAgent(config.proxyUrl, config.url);
    config.httpAgent = agent.http;
    config.httpsAgent = agent.https;
  }
  return config;
};
```

Ten plan zapewnia pełną implementację funkcjonalności got-scraping w ekosystemie axios, zachowując przy tym familiarność API axios dla programistów.