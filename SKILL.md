---
name: tracking-essencial
description: "Fundamentos de rastreamento digital para Claude Code: Meta Pixel e Conversions API (CAPI) com deduplicação por event_id, GA4 e Measurement Protocol, Google Tag Manager, data layer, server-side, UTMs e atribuição, identity resolution, webhooks, BigQuery, LGPD, QA, Google Ads, TikTok Events API, validação com Zod e monitoramento. Use quando o pedido envolver pixel, CAPI, GA4, GTM, dataLayer, event_id, _fbp, _fbc, UTM, atribuição, conversão, server-side ou auditoria de tracking."
metadata:
  author: tracking-essencial
  version: "1.0.0"
---

# Tracking Essencial: engenharia de dados para mídia paga

Você é um engenheiro de dados de rastreamento digital sênior. Conhece a fundo Meta Pixel, CAPI, GA4, GTM, server-side, pipelines de dados, BigQuery e LGPD. Quando acionado, aplique este conhecimento para entregar implementações corretas, robustas e auditáveis.

### Como ler esta skill

Ela é **neutra de fornecedor e de projeto**. Todo identificador nos exemplos é placeholder; substitua pelos do seu ambiente:

| Placeholder | O que é |
|---|---|
| `GTM-XXXXXXX` · `G-XXXXXXXXXX` · `AW-1234567890` | container web, propriedade GA4, conta Google Ads |
| `SEUSITE.com.br` · `SERVIDOR` · `HOST` | domínio do site e do tagging server |
| `SEU_PROJETO` · `REGIAO` | projeto e região no provedor de nuvem |
| `Kit Inicial` · `acme_*` | produto e campanha de exemplo |

Esta é a versão **essencial** (seções 1 a 20: fundamentos e implementação). A versão completa traz mais 31 seções de casos reais em produção (ver README).

---

## 1. Filosofia e Princípios

**1. Data Quality First.** Dado errado é pior que dado ausente. Valide antes de enviar. Uma conversão duplicada distorce bid estratégico e desperdiça budget.

**2. Privacy by Design.** Colete só o necessário. Hash PII antes de enviar a terceiros. Documente o que é coletado e por quê. Nunca envie dados não-anonimizados para plataformas de ads.

**3. Single Source of Truth.** Defina qual plataforma é autoritativa para cada métrica. Divergências entre GA4, Meta e CRM são normais: o que importa é saber qual confiar para cada decisão.

**4. Deduplicação sempre.** Toda implementação Pixel + CAPI **deve** ter `event_id` coincidente. Toda implementação GA4 + Measurement Protocol **deve** ter `client_id` e `session_id` coincidentes.

**5. Observabilidade.** Cada evento enviado deve ser logado server-side com `event_id`, `event_name`, `timestamp` e status da resposta da API. Sem log, não há diagnóstico.

---

## 2. Taxonomia de Eventos: Design Canônico

### 2.1 Convenções de nomenclatura

| Plataforma | Convenção | Exemplo |
|---|---|---|
| Meta Pixel / CAPI | PascalCase (eventos padrão) | `Lead`, `Purchase`, `Contact`, `ViewContent` |
| GA4 | snake_case | `form_submit`, `button_click`, `purchase` |
| Amplitude/Mixpanel | `[Object] [Action]` | `Form Submitted`, `Button Clicked` |
| GTM triggers internos | `event_name` no dataLayer | `lead_form_submit` |

### 2.2 Hierarquia de eventos

```
Sessão
 └── PageView                    (toda navegação de página)
      └── ViewContent            (produto/conteúdo visto)
           └── InitiateCheckout  (início de checkout)
           └── AddToCart         (item adicionado)
           └── Lead              (formulário submetido)
                └── Purchase     (compra finalizada)
```

### 2.3 Parâmetros obrigatórios por evento

```typescript
// Mínimo universal: todo evento deve ter estes campos
interface BaseEvent {
  event_id: string;        // UUID v4: único por disparo
  event_name: string;      // Nome canônico da plataforma
  event_time: number;      // Unix timestamp em segundos
  event_source_url: string; // URL completa com query string
}

// User data: máxima correspondência aumenta taxa de match
interface UserData {
  em?: string[];           // email SHA-256 normalizado
  ph?: string[];           // telefone SHA-256 (só dígitos, com DDI)
  fn?: string[];           // primeiro nome SHA-256 normalizado
  ln?: string[];           // sobrenome SHA-256 normalizado
  ct?: string[];           // cidade SHA-256 (lowercase, sem acento)
  st?: string[];           // estado 2 letras SHA-256
  zp?: string[];           // CEP SHA-256
  country?: string[];      // ISO 2 letras SHA-256: sempre incluir
  external_id?: string[];  // ID interno SHA-256: chave de identidade
  client_ip_address: string;  // NÃO hashar
  client_user_agent: string;  // NÃO hashar
  fbp?: string;            // cookie _fbp: NÃO hashar
  fbc?: string;            // cookie _fbc: NÃO hashar
}
```

---

## 3. Meta Pixel & Conversions API (CAPI)

### 3.1 Arquitetura de deduplicação

```
Browser                          Servidor                    Meta
  │                                 │                          │
  ├─ fbq('track','Lead',data,       │                          │
  │      {eventID: uuid})  ─────────┼──────────────────────────► pixel event
  │                                 │                          │
  ├─ POST /api/leads ──────────────►│                          │
  │   { tracking.event_id: uuid }   │                          │
  │                                 ├─ CAPI event_id=uuid ─────► server event
  │                                 │                          │
  │                                 │              deduplica por event_id ✓
```

**Regra crítica:** O `event_id` deve ser gerado **uma única vez** no cliente, antes de qualquer disparo, e enviado identicamente para pixel e CAPI.

### 3.2 Normalização PII para SHA-256 (obrigatória)

```typescript
import crypto from "crypto";

const sha256 = (v: string) => crypto.createHash("sha256").update(v).digest("hex");
const stripAccents = (s: string) => s.normalize("NFD").replace(/[̀-ͯ]/g, "");

// Cada campo tem sua própria normalização: não use sha256 direto
const normalize = {
  email:   (v: string) => v ? sha256(v.trim().toLowerCase()) : "",
  phone:   (v: string) => { const d = v.replace(/\D/g, ""); return d ? sha256(d) : ""; },
  name:    (v: string) => { const n = stripAccents(v.trim().toLowerCase()).replace(/[^a-z0-9]/g, ""); return n ? sha256(n) : ""; },
  city:    (v: string) => { const n = stripAccents(v.trim().toLowerCase()).replace(/[^a-z]/g, ""); return n ? sha256(n) : ""; },
  state:   (v: string) => { const n = stripAccents(v.trim().toLowerCase()).replace(/[^a-z]/g, "").slice(0,2); return n ? sha256(n) : ""; },
  country: (v: string) => { const n = v.trim().toLowerCase().slice(0,2); return n ? sha256(n) : ""; },
  zip:     (v: string) => { const n = v.trim().toLowerCase().replace(/\s+/g,""); return n ? sha256(n) : ""; },
  extId:   (v: string) => v ? sha256(v.trim()) : "",
};
```

### 3.3 Eventos padrão Meta e seus parâmetros

| Evento | Quando disparar | Parâmetros obrigatórios |
|---|---|---|
| `PageView` | Toda página carregada | `event_source_url` |
| `ViewContent` | Produto/página de conteúdo vista | `content_ids`, `content_type`, `value`, `currency` |
| `Search` | Busca realizada | `search_string` |
| `AddToCart` | Item adicionado ao carrinho | `content_ids`, `content_type`, `value`, `currency` |
| `InitiateCheckout` | Início do processo de checkout | `value`, `currency`, `num_items` |
| `AddPaymentInfo` | Dados de pagamento inseridos | `value`, `currency` |
| `Lead` | Formulário de captação enviado | `content_name` (opcional) |
| `CompleteRegistration` | Cadastro concluído | `status` |
| `Contact` | Clique em WhatsApp, telefone, chat | |
| `Purchase` | Compra finalizada | `value`, `currency`, `content_ids`, `content_type` |
| `Subscribe` | Assinatura de newsletter/serviço | `value`, `currency`, `predicted_ltv` |

### 3.4 ITP Bypass: cookies first-party via servidor

```typescript
// Middleware Express: seta _fbp/_fbc via Set-Cookie HTTP
// Cookies HTTP não sofrem cap de 7 dias do Safari ITP/iOS
app.use((req, res, next) => {
  if (req.method !== "GET" || /\.(js|css|webp|png|ico|json)$/i.test(req.path)) return next();

  const cookies = parseCookies(req.headers.cookie);
  const toSet: string[] = [];
  const MAX_AGE = 60 * 60 * 24 * 90; // 90 dias

  if (!cookies._fbp) {
    toSet.push(`_fbp=fb.1.${Date.now()}.${Math.floor(Math.random()*9_999_999_999)}; Path=/; Max-Age=${MAX_AGE}; SameSite=Lax; Secure`);
  }
  const fbclid = req.query.fbclid as string;
  if (!cookies._fbc && fbclid) {
    toSet.push(`_fbc=fb.1.${Date.now()}.${fbclid}; Path=/; Max-Age=${MAX_AGE}; SameSite=Lax; Secure`);
  }
  if (toSet.length) res.setHeader("Set-Cookie", toSet);
  next();
});
```

### 3.5 Diagnóstico de erros CAPI comuns

| Código | Mensagem | Causa | Solução |
|---|---|---|---|
| 190 | Invalid OAuth access token | Token expirado ou inválido | Gerar novo System User Token |
| 100 | Invalid parameter | Campo obrigatório ausente ou formato errado | Verificar `action_source`, `event_time` |
| 2650 | Event not deduplicated | `event_id` não bate entre pixel e CAPI | Garantir mesmo UUID nos dois disparos |
| 2652 | Data processing error | PII não normalizada corretamente | Verificar pipeline de hash SHA-256 |
| #132000 | Processed params format | Estrutura do payload incorreta | Checar separação header/body nos params |

### 3.6 Payload CAPI completo (template)

```typescript
const capiPayload = {
  data: [{
    event_name: "Lead",               // string exata Meta
    event_time: Math.floor(Date.now() / 1000), // unix seconds
    event_id: crypto.randomUUID(),    // UUID v4: igual ao pixel
    event_source_url: "https://...",
    action_source: "website",         // sempre "website" para web
    user_data: {
      em: [normalize.email(email)],
      ph: [normalize.phone(phone)],
      fn: [normalize.name(firstName)],
      ln: [normalize.name(lastName)],
      ct: [normalize.city(city)],
      st: [normalize.state(state)],
      country: [normalize.country("BR")],
      external_id: [normalize.extId(userId)],
      client_ip_address: clientIp,    // raw: não hashar
      client_user_agent: userAgent,   // raw: não hashar
      fbp: cookieFbp || undefined,    // raw: não hashar
      fbc: cookieFbc || undefined,    // raw: não hashar
    },
    custom_data: {
      content_name: "Kit Inicial",
      content_category: "Revenda",
      currency: "BRL",
      value: 297.00,
    }
  }],
  // test_event_code: "TEST12345", remover em produção!
};
```

---

## 4. Google Analytics 4 (GA4)

### 4.1 Eventos recomendados (e-commerce)

```javascript
// gtag.js: padrão para SPA
gtag('event', 'view_item', {
  currency: 'BRL',
  value: 297.00,
  items: [{ item_id: 'kit-ini-001', item_name: 'Kit Inicial', price: 297.00, quantity: 1 }]
});

gtag('event', 'generate_lead', {
  currency: 'BRL',
  value: 0,
  // custom params ficam como custom dimensions/metrics no GA4
  form_id: 'lead-form-hero',
  kit_selected: 'Kit Inicial',
});

gtag('event', 'purchase', {
  transaction_id: 'ORDER-123',   // obrigatório: usado para deduplicação
  value: 297.00,
  tax: 0,
  shipping: 0,
  currency: 'BRL',
  items: [{ item_id: 'kit-ini-001', item_name: 'Kit Inicial', price: 297.00, quantity: 1 }]
});
```

### 4.2 Measurement Protocol GA4 (server-side)

```typescript
// Deduplicação: client_id e session_id devem vir do cookie _ga e _ga_XXXXXX
const GA4_MEASUREMENT_ID = process.env.GA4_MEASUREMENT_ID; // G-XXXXXXXXXX
const GA4_API_SECRET     = process.env.GA4_API_SECRET;

async function sendGA4Event(clientId: string, sessionId: string, eventName: string, params: object) {
  await fetch(
    `https://www.google-analytics.com/mp/collect?measurement_id=${GA4_MEASUREMENT_ID}&api_secret=${GA4_API_SECRET}`,
    {
      method: "POST",
      body: JSON.stringify({
        client_id: clientId,           // valor do cookie _ga (sem "GA1.1.")
        // user_id: userId,            // opcional: para cross-device
        events: [{
          name: eventName,
          params: {
            session_id: sessionId,     // valor do cookie _ga_XXXXXXXX
            engagement_time_msec: 1,   // mínimo para contar como sessão engajada
            ...params,
          }
        }]
      })
    }
  );
}
```

### 4.3 Extração do client_id e session_id dos cookies

```typescript
// _ga = GA1.1.XXXXXXXXXX.YYYYYYYYYY → client_id = XXXXXXXXXX.YYYYYYYYYY
const getGA4ClientId = (cookieHeader?: string): string => {
  const match = cookieHeader?.match(/_ga=GA\d\.\d\.(\d+\.\d+)/);
  return match?.[1] || "";
};

// _ga_XXXXXXXXXX = GS1.1.TIMESTAMP.COUNT.IS_FIRST.LAST_EVENT.SESSION_ID...
const getGA4SessionId = (cookieHeader: string, measurementId: string): string => {
  const key = `_ga_${measurementId.replace("G-", "")}`;
  const match = cookieHeader.match(new RegExp(`${key}=GS\\d\\.\\d\\.(\\d+)`));
  return match?.[1] || "";
};
```

---

## 5. Google Tag Manager (GTM)

### 5.1 Arquitetura de Data Layer

```javascript
// SEMPRE inicializar antes do GTM script
window.dataLayer = window.dataLayer || [];

// Padrão: push event com todos os dados relevantes
// Nunca empurrar dados sensíveis (CPF, senha, cartão)
window.dataLayer.push({
  event: 'lead_form_submit',      // nome interno GTM
  // Context
  page_type: 'landing',
  page_url: window.location.href,
  // Lead data
  lead_kit: 'Kit Inicial',
  lead_investment: 'R$ 300 a R$ 600',
  lead_channel: ['Instagram', 'WhatsApp'],
  // Attribution
  fbclid: new URLSearchParams(location.search).get('fbclid') || undefined,
  // Identity (nunca PII raw)
  external_id: localStorage.getItem('app_external_id') || undefined,
});
```

### 5.2 Convenções de variáveis GTM

| Tipo | Prefixo | Exemplo |
|---|---|---|
| Data Layer Variable | `dlv_` | `dlv_lead_kit` |
| JavaScript Variable | `jsv_` | `jsv_page_type` |
| 1st Party Cookie | `ck_` | `ck_external_id` |
| Lookup Table | `lt_` | `lt_kit_to_value` |
| GA4 Config Tag | tag sem prefixo | `GA4 - Config` |
| Meta Pixel Tag | tag sem prefixo | `Meta Pixel - Lead` |
| Trigger | ação + objeto | `Click - WhatsApp Button` |

### 5.3 SPA tracking no GTM

```javascript
// React Router / Next.js: re-dispara PageView no pushState
const originalPush = history.pushState.bind(history);
history.pushState = (...args) => {
  originalPush(...args);
  window.dataLayer.push({ event: 'spa_page_view', page_url: location.href });
};
window.addEventListener('popstate', () => {
  window.dataLayer.push({ event: 'spa_page_view', page_url: location.href });
});
```

### 5.4 Conversões otimizadas do Google Ads no GTM

Apurado contra dois contêineres reais em 01/09/2026, um funcionando e um recém-montado.
Não deduza esta seção: ela contraria o que a interface do GTM sugere.

**A tag de dados do usuário é uma tag própria, não um campo da tag de conversão.**
Tipo: **Google Ads > Google Ads User-provided Data Event** (`__awud` no `gtm.js`).
Campos: `ID de conversão` e `Dados fornecidos pelo usuário` (uma variável do tipo
"Dados fornecidos pelo usuário", `__awec`, modo manual, com e-mail e telefone).
Gatilho: All Pages. Quando o dado só entra no dataLayer no momento do envio,
acrescente também o gatilho do evento de conversão.

**O `ID de conversão` é o número puro, sem o prefixo `AW-`.**
Vale para a tag de conversão (`__awct`), para a `__awud` e para a variável constante
que as alimenta. O prefixo `AW-` só aparece no `ID da tag` da **Tag do Google**
(`__googtag`), que é outra coisa. Com `AW-` na variável, a interface mostra o
destino como `AW-AW-1234567890` e o GTM passa a enxergar **dois destinos distintos**:
a conversão não casa com a Tag do Google e o aviso "Nenhuma tag do Google foi
encontrada neste contêiner" nunca sai. Corrigido o prefixo, o aviso vira verde:
"Esta tag vai usar a configuração da tag do Google <nome>".

**Caminhos que parecem certos e não são:**
- Parâmetro de evento `user_data` na tag de conversão. Um artigo de suporte do Google
  descreve isso para a *Tag do Google*, não para a tag de conversão, e o contêiner que
  funciona não usa. Use a `__awud`.
- As caixas "Fornecer dados de vendas no nível do produto", "Fornecer novos dados do
  cliente" e "Fornecer os dados para envio" na tag de conversão. As três são
  e-commerce (produto, aquisição de novo cliente e **frete**). Nenhuma tem relação com
  conversões otimizadas.

**Nunca use os assistentes automáticos do GTM.** O botão "Criar tag" ao lado do aviso
de tag do Google faltando, e a "configuração guiada do Google Ads", preenchem campos
sozinhos e erram: num caso real criaram uma Tag do Google **nomeada** com o ID certo e
**configurada** com o ID de conversão de OUTRO cliente da mesma agência, e ainda
reescreveram a variável de ID de conversão. Foi ao ar antes de ser pego.

**Validação obrigatória depois de publicar**: a tela mente, o arquivo servido não:

```bash
curl -s "https://www.googletagmanager.com/gtm.js?id=GTM-XXXXXXX" -o c.js
grep -o "AW-AW-" c.js | wc -l          # tem que ser 0
grep -o "AW-[0-9]*" c.js | sort -u     # só os IDs deste cliente
node -e "const s=require('fs').readFileSync('c.js','utf8');
  console.log(s.match(/\"tags\":\[(.*?)\],\n  \"predicates\"/s)[1]
    .match(/\"function\":\"(__[a-z0-9_]+)\"/g).join(', '))"
```

Espere ver `__awct` (uma por conversão), `__awud` (uma), `__googtag` e `__gclidw`.
Se `__awud` não aparecer, as conversões otimizadas estão desligadas por mais que a
conta diga que estão ativas: o mecanismo fica ligado e sem combustível.

**Método para montar contêiner de cliente novo:** exportar um contêiner que já
funciona, importar, trocar as variáveis `const - *` e apagar o que não se aplica.
Montar tag por tag pela interface reintroduz todos os erros acima.

---

## 6. Server-Side Tracking

### 6.1 Decisão: Browser Pixel vs CAPI vs sGTM

| Cenário | Recomendação |
|---|---|
| Eventos de conversão críticos (Lead, Purchase) | CAPI obrigatório + Pixel para deduplicar |
| Eventos de engajamento (scroll, clique) | Pixel suficiente |
| Dados sensíveis (endereço, CPF) | CAPI exclusivamente: nunca browser |
| Safari/iOS (ITP) | Server-side cookies + CAPI |
| Ad blocker presente | CAPI é a única forma de capturar |
| Latência crítica | Pixel primeiro, CAPI async |

### 6.2 Ordem de operações para deduplicação correta

```
❌ ERRADO: pixel pode disparar sem par CAPI (rede, erro, retry)
  1. Gera event_id
  2. Dispara pixel (browser)
  3. Chama servidor → falha → retry com NOVO event_id → pixel disparou 2x

✅ CORRETO para formulários (sem risco de popup blocker)
  1. Gera event_id
  2. POST servidor → CAPI dispara server-side
  3. Servidor retorna 200
  4. Pixel dispara browser (com mesmo event_id)
  → Ambos disparam juntos ou nenhum dispara

✅ CORRETO para cliques em links (abre nova aba: página continua viva)
  1. Gera event_id
  2. Inicia fetch servidor (fire-and-forget, sem await)
  3. Dispara pixel (browser)
  4. Navegação acontece normalmente
  → Risco mínimo de orphaned event (não há retry de clique)
```

### 6.3 Endpoint CAPI genérico (Express)

```typescript
const sendMetaCapi = async (events: CAPIEvent[]) => {
  const { META_PIXEL_ID, META_ACCESS_TOKEN, META_TEST_EVENT_CODE } = process.env;
  if (!META_PIXEL_ID || !META_ACCESS_TOKEN) {
    console.warn("[CAPI] Variáveis META_PIXEL_ID ou META_ACCESS_TOKEN não configuradas");
    return;
  }

  const payload: Record<string, any> = { data: events };
  if (META_TEST_EVENT_CODE) payload.test_event_code = META_TEST_EVENT_CODE;

  try {
    const res = await axios.post(
      `https://graph.facebook.com/v21.0/${META_PIXEL_ID}/events?access_token=${META_ACCESS_TOKEN}`,
      payload
    );
    console.log(`[CAPI] OK events=${events.map(e=>e.event_name).join(',')} ids=${events.map(e=>e.event_id).join(',')}`, res.data);
  } catch (err: any) {
    console.error(`[CAPI] Error`, err.response?.status, JSON.stringify(err.response?.data));
    // NÃO relançar: falha de CAPI não deve afetar o fluxo principal
  }
};
```

---

## 7. Atribuição e UTM

### 7.1 Taxonomia UTM padrão

```
utm_source    → tráfego de origem (facebook, google, instagram, whatsapp)
utm_medium    → canal de mídia (cpc, cpm, email, organic, referral, social)
utm_campaign  → nome da campanha (acme_lead_kit-inicial_maio2026)
utm_content   → variação criativa (video-depoimento_v1, carrossel_v2)
utm_term      → palavra-chave (revenda-atacado, atacado-infantil)
```

**Convenção de nomenclatura:** `{produto}_{objetivo}_{audiencia}_{período}`
Exemplo: `acme_lead_lookalike-30d_maio2026`

### 7.2 Preservação de atribuição entre páginas (SPA)

```typescript
// Persiste UTMs no sessionStorage ao chegar na página
const persistUTMs = () => {
  const params = new URLSearchParams(location.search);
  const utmKeys = ['utm_source','utm_medium','utm_campaign','utm_content','utm_term','fbclid','gclid'];
  const stored: Record<string, string> = {};

  utmKeys.forEach(k => {
    const v = params.get(k);
    if (v) stored[k] = v;
  });

  if (Object.keys(stored).length) {
    sessionStorage.setItem('attribution', JSON.stringify(stored));
  }
};

// Recupera na conversão
const getAttribution = (): Record<string, string> => {
  try { return JSON.parse(sessionStorage.getItem('attribution') || '{}'); } catch { return {}; }
};
```

### 7.3 Modelos de atribuição

| Modelo | Quando usar | Limitação |
|---|---|---|
| Last Click | Default Meta/Google | Subestima awareness, sobrestima retargeting |
| First Click | Análise de aquisição | Ignora jornada de maturação |
| Linear | Funis longos (B2B) | Dilui o peso dos pontos críticos |
| Data-Driven | Volume > 300 conversões/30d | Requer dados suficientes |
| Time Decay | Ciclo de venda rápido | Penaliza campanhas de awareness |

**Regra prática:** Use Data-Driven quando disponível. Se volume insuficiente, use Last Click com janelas longas (7d click + 1d view para Meta).

### 7.4 Meta Lookback Windows: configuração correta

```
Janela de clique: 7 dias  (padrão)  → adequado para ciclos < 7d
                  1 dia             → ciclos de compra muito curtos (impulso)

Janela de view:   1 dia   (padrão)  → para campanhas de awareness/vídeo
                  Desativada        → quando quer contar só intenção direta

Configurar em: Meta Ads → Conjunto de Anúncios → Atribuição → Janela de atribuição

ATENÇÃO: janela de view gera volume alto de conversões atribuídas mas baixa
qualidade: compare ROAS com e sem view-through antes de escalar.
```

**Lookback window no CAPI:** o campo `event_time` deve ser o timestamp real do evento, não `Date.now()` atrasado. Se o evento for processado horas depois (batch), use o timestamp original. Eventos com `event_time` > 7 dias são rejeitados pela API.

### 7.5 Cross-domain tracking (GA4)

```javascript
// Configura domínios que compartilham session_id via parâmetro _gl na URL
// Necessário quando a jornada atravessa subdomínios ou domínios distintos
gtag('config', 'G-XXXXXXXXXX', {
  linker: {
    domains: ['loja.seusite.com.br', 'checkout.seusite.com.br', 'seusite.com.br'],
    accept_incoming: true,   // aceita _gl recebido de domínios listados
  }
});

// Para GTM: usar tag "GA4 Configuration" com campo "Domains to Link" preenchido
// O parâmetro _gl é adicionado automaticamente nos links entre os domínios
```

---

## 8. Identity Resolution

### 8.1 Identifiers por camada

```
Nível         Identifier           Persistência    Limitações
─────────────────────────────────────────────────────────────────
Dispositivo   _fbp, _ga            90d (HTTP)      Por dispositivo
Sessão        session_id           Sessão          Expira ao fechar
Usuário auth  user_id / external_id Permanente     Só logados
Cross-device  email hash           Permanente      Só com PII
Anônimo       localStorage UUID    Até limpar      Sem cross-device
```

### 8.2 External ID: geração e uso

```typescript
// Gera uma vez, persiste em localStorage, envia em TODOS os eventos
const getOrCreateExternalId = (): string => {
  const KEY = 'app_external_id'; // use prefixo único por app
  let id = localStorage.getItem(KEY);
  if (!id) {
    id = crypto.randomUUID();
    localStorage.setItem(KEY, id);
  }
  return id;
};

// Inicializa pixel com external_id para melhorar taxa de match
fbq('init', PIXEL_ID, { external_id: getOrCreateExternalId() });

// Envia também na CAPI (hashed)
user_data.external_id = [sha256(externalId.trim())];
```

---

## 9. Data Layer Schema: Padrão Canônico

```typescript
// Schema base para todos os projetos
interface DataLayerEvent {
  // Obrigatório em todo push
  event: string;                    // nome do trigger GTM

  // Contexto de página
  page_type?: 'home' | 'landing' | 'product' | 'checkout' | 'confirmation';
  page_url?: string;
  page_title?: string;

  // Identidade (nunca PII raw)
  external_id?: string;             // UUID do localStorage
  user_logged_in?: boolean;

  // Produto/conteúdo
  item_id?: string;
  item_name?: string;
  item_category?: string;
  item_price?: number;
  currency?: string;

  // Conversão
  transaction_id?: string;          // para Purchase: deduplicação GA4
  value?: number;

  // Atribuição
  utm_source?: string;
  utm_medium?: string;
  utm_campaign?: string;
  fbclid?: string;
  gclid?: string;

  // Formulário
  form_id?: string;
  form_name?: string;
}
```

---

## 10. Pipelines de Dados e Webhooks

### 10.1 Webhook design: contrato mínimo

```typescript
// Todo webhook deve ter:
interface WebhookPayload {
  // Metadata de entrega
  webhook_id: string;         // UUID único desta entrega
  webhook_version: string;    // "2.0", para evolução sem breaking
  event_type: string;         // "lead.created", "purchase.completed"
  created_at: string;         // ISO 8601

  // Dados do evento
  data: Record<string, any>;

  // Assinatura HMAC para validação (segurança)
  signature: string;          // HMAC-SHA256 do body com secret
}

// Validação no receptor (Express)
const validateWebhook = (rawBody: Buffer, signature: string, secret: string): boolean => {
  const expected = crypto.createHmac('sha256', secret).update(rawBody).digest('hex');
  return crypto.timingSafeEqual(Buffer.from(signature), Buffer.from(expected));
};
```

### 10.2 Padrão de retry para eventos críticos

```typescript
// Exponential backoff para CAPI e webhooks críticos
const sendWithRetry = async (fn: () => Promise<void>, maxAttempts = 3) => {
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      await fn();
      return;
    } catch (err) {
      if (attempt === maxAttempts) throw err;
      await new Promise(r => setTimeout(r, 2 ** attempt * 500)); // 1s, 2s, 4s
    }
  }
};
```

### 10.3 Schema evolution: versionamento de eventos

```
v1: { name, phone }                           → campo "name" completo
v2: { first_name, last_name, phone }          → breaking change: incrementar versão
v2.1: { first_name, last_name, phone, email } → adição compatível: minor

Regras:
  - NUNCA remover campo sem deprecate de 90 dias
  - NUNCA mudar tipo de um campo existente
  - Campos novos SEMPRE opcionais (backward compatible)
  - Usar versão semântica: major.minor.patch
```

---

## 11. BigQuery & SQL Analítico (GA4)

### 11.1 Estrutura da tabela GA4 no BigQuery

```sql
-- Tabela: events_YYYYMMDD ou events_intraday_YYYYMMDD
-- Particionada por data, clusterizada por event_name

-- Query padrão: extrair parâmetros de evento
SELECT
  event_date,
  event_name,
  event_timestamp,
  user_pseudo_id,                                           -- client_id
  (SELECT value.int_value  FROM UNNEST(event_params) WHERE key = 'ga_session_id')  AS session_id,
  (SELECT value.string_value FROM UNNEST(event_params) WHERE key = 'page_location') AS page_url,
  (SELECT value.string_value FROM UNNEST(event_params) WHERE key = 'source')        AS source,
  (SELECT value.string_value FROM UNNEST(event_params) WHERE key = 'medium')        AS medium,
  (SELECT value.string_value FROM UNNEST(event_params) WHERE key = 'campaign')      AS campaign,
  geo.country,
  geo.region,
  device.category                                          -- desktop, mobile, tablet
FROM `project.dataset.events_*`
WHERE _TABLE_SUFFIX BETWEEN '20260101' AND '20260516'
  AND event_name IN ('generate_lead', 'purchase', 'view_item')
```

### 11.2 Análise de funil (SQL)

```sql
-- Funil: PageView → ViewContent → Lead
WITH user_events AS (
  SELECT
    user_pseudo_id,
    MAX(CASE WHEN event_name = 'page_view'     THEN 1 ELSE 0 END) AS reached_pageview,
    MAX(CASE WHEN event_name = 'view_item'     THEN 1 ELSE 0 END) AS reached_view_content,
    MAX(CASE WHEN event_name = 'generate_lead' THEN 1 ELSE 0 END) AS reached_lead
  FROM `project.dataset.events_*`
  WHERE _TABLE_SUFFIX BETWEEN '20260401' AND '20260516'
  GROUP BY user_pseudo_id
)
SELECT
  COUNTIF(reached_pageview = 1)                                   AS step1_pageview,
  COUNTIF(reached_view_content = 1)                               AS step2_view_content,
  COUNTIF(reached_lead = 1)                                       AS step3_lead,
  ROUND(COUNTIF(reached_view_content = 1) / COUNTIF(reached_pageview = 1) * 100, 2) AS cvr_pv_to_vc,
  ROUND(COUNTIF(reached_lead = 1) / COUNTIF(reached_view_content = 1) * 100, 2)    AS cvr_vc_to_lead,
  ROUND(COUNTIF(reached_lead = 1) / COUNTIF(reached_pageview = 1) * 100, 2)        AS cvr_overall
FROM user_events
```

### 11.3 Detecção de duplicatas (auditoria CAPI)

```sql
-- Identifica event_ids duplicados: indica deduplicação falhando
SELECT
  event_id,
  event_name,
  COUNT(*) AS ocurrencias,
  MIN(event_time) AS primeiro_disparo,
  MAX(event_time) AS ultimo_disparo,
  TIMESTAMP_DIFF(TIMESTAMP_SECONDS(MAX(event_time)), TIMESTAMP_SECONDS(MIN(event_time)), SECOND) AS gap_segundos
FROM `project.dataset.capi_events_log`  -- sua tabela de log de CAPI
WHERE DATE(TIMESTAMP_SECONDS(event_time)) >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)
GROUP BY event_id, event_name
HAVING COUNT(*) > 1
ORDER BY ocurrencias DESC
```

---

## 12. Compliance: LGPD / GDPR

### 12.1 Classificação de dados por sensibilidade

| Dado | Classificação | Tratamento |
|---|---|---|
| Email, telefone, nome | PII direta | Hash SHA-256 antes de enviar a terceiros |
| IP, user agent, device ID | PII indireta | Enviar raw somente para CAPI (necessário para matching) |
| Comportamento navegação | Dado comportamental | Anonimizar após 90 dias |
| Preferências, interesse | Dado inferido | Não compartilhar sem consentimento explícito |
| CPF, RG, data de nascimento | PII sensível | Nunca enviar para plataformas de ads |

### 12.2 Consent Mode v2 (Google)

```javascript
// DEVE ser definido ANTES do GTM/gtag
window.dataLayer = window.dataLayer || [];
function gtag(){ dataLayer.push(arguments); }

// Estado padrão = negado (privacy-first)
gtag('consent', 'default', {
  analytics_storage:     'denied',   // GA4
  ad_storage:            'denied',   // Google Ads
  ad_user_data:          'denied',   // dados de usuário para ads
  ad_personalization:    'denied',   // personalização
  functionality_storage: 'granted',  // cookies essenciais
  wait_for_update:       500,        // ms para CMP carregar
});

// Após usuário aceitar via CMP:
gtag('consent', 'update', {
  analytics_storage:  'granted',
  ad_storage:         'granted',
  ad_user_data:       'granted',
  ad_personalization: 'granted',
});
```

### 12.3 Checklist de compliance antes de go-live

- [ ] CMP implementado com opção de recusar (não só aceitar)
- [ ] Cookies não essenciais bloqueados antes do consentimento
- [ ] PII hashada antes de qualquer envio a terceiros
- [ ] Aviso de coleta de dados na política de privacidade
- [ ] Retenção de dados definida (GA4: 14 meses máx, BigQuery: conforme política)
- [ ] Mecanismo de exclusão de dados do usuário disponível
- [ ] `META_TEST_EVENT_CODE` removido do ambiente de produção
- [ ] Nenhum dado de menores de 13 anos coletado sem consentimento parental

---

## 13. Debugging e QA

### 13.0 React StrictMode: gotcha de eventos duplicados

Em desenvolvimento, React StrictMode invoca efeitos duas vezes propositalmente.
`useEffect` que dispara eventos de tracking vai duplicar em dev **mas não em produção**.

```typescript
// ❌ Vai duplicar em dev (StrictMode double-invoke)
useEffect(() => {
  fbq('track', 'ViewContent', { content_name: product.name });
}, []);

// ✅ Guard com ref para disparar só uma vez mesmo em StrictMode
const tracked = useRef(false);
useEffect(() => {
  if (tracked.current) return;
  tracked.current = true;
  fbq('track', 'ViewContent', { content_name: product.name });
}, []);

// ✅ Alternativa: disparar no evento de usuário (click, submit), não em efeito
// Eventos de usuário nunca são duplicados pelo StrictMode
```

**Regra:** nunca disparar conversões em `useEffect` sem guard de ref. Use eventos de usuário (`onClick`, `onSubmit`) sempre que possível.

### 13.1 Ferramentas por plataforma

| Ferramenta | Plataforma | O que verifica |
|---|---|---|
| Meta Pixel Helper (Chrome ext) | Meta Pixel | Pixel iniciado, eventos disparados, parâmetros |
| Meta Events Manager | Meta CAPI | Recebimento CAPI, taxa de match, deduplicação |
| GA4 DebugView | GA4 | Eventos em tempo real, parâmetros, conversões |
| GTM Preview Mode | GTM | Triggers disparando, variáveis resolvidas, tags |
| GA4 Realtime | GA4 | Sessões ativas, eventos ao vivo |
| Network Tab (DevTools) | Qualquer | Payload exato enviado, status HTTP |
| `console.log(dataLayer)` | GTM | Estado atual do dataLayer |

### 13.2 Checklist de QA de rastreamento

**Pixel / CAPI:**
- [ ] `event_id` UUID no pixel bate com `event_id` no CAPI (verificar nos logs do servidor)
- [ ] `META_PIXEL_ID` env var igual ao ID no `fbq('init', ...)`
- [ ] `META_TEST_EVENT_CODE` ativo apenas em staging
- [ ] Todos os campos PII hashados corretamente (testar com valor conhecido)
- [ ] `_fbp` e `_fbc` sendo enviados em `user_data`
- [ ] Taxa de correspondência > 70% no Events Manager

**GA4:**
- [ ] `client_id` correto no Measurement Protocol (sem "GA1.1." prefix)
- [ ] `transaction_id` único por compra (deduplicação GA4)
- [ ] Nenhum PII em parâmetros de evento (nome, email, telefone)
- [ ] Custom dimensions registradas no painel GA4

**Geral:**
- [ ] Nenhum evento disparando no carregamento quando não deveria
- [ ] Sem eventos duplicados em SPAs (verificar re-renders)
- [ ] Eventos de conversão testados em todos os fluxos (desktop + mobile)
- [ ] React StrictMode: verificar que eventos de conversão não duplicam em dev

### 13.3 Filtro de bots e tráfego interno

Sem filtragem, bots e acessos internos da equipe inflam conversões e distorcem o modelo de bid da plataforma.

```typescript
// Filtragem server-side antes de enviar à CAPI
const INTERNAL_IPS = (process.env.INTERNAL_IPS || "").split(",").filter(Boolean);
const BOT_UA_PATTERNS = [
  /googlebot/i, /bingbot/i, /slurp/i, /duckduckbot/i,
  /baiduspider/i, /yandexbot/i, /sogou/i, /exabot/i,
  /facebot/i, /ia_archiver/i, /semrushbot/i, /ahrefsbot/i,
  /mj12bot/i, /dotbot/i, /petalbot/i,
];

const isBot = (userAgent: string): boolean =>
  BOT_UA_PATTERNS.some(p => p.test(userAgent));

const isInternalIP = (ip: string): boolean =>
  INTERNAL_IPS.includes(ip) || ip.startsWith("127.") || ip === "::1";

// No endpoint /api/leads:
const clientIp = req.headers["x-forwarded-for"]?.split(",")[0].trim() || req.ip || "";
const ua = req.headers["user-agent"] || "";

if (isBot(ua) || isInternalIP(clientIp)) {
  console.log(`[CAPI] Evento ignorado: bot/IP interno: ip=${clientIp} ua=${ua}`);
  return res.status(204).end(); // responde OK mas não envia à CAPI
}
```

**No GA4:** em Admin → Data Filters → criar filtro "Internal Traffic" com os IPs internos e definir como "Active" (não só "Testing"). Sem isso, sessões internas contam nas métricas.

### 13.4 Verificação de deduplicação em produção

```bash
# Agnostik de plataforma: qualquer sistema com logs estruturados
# Buscar pelo padrão que o código loga: "[CAPI] Lead event_id=..."

# Cloud Run (GCP)
gcloud logging read 'textPayload=~"\[CAPI\]"' --project=SEU_PROJETO --limit=20

# Docker / qualquer servidor
docker logs CONTAINER_NAME 2>&1 | grep "\[CAPI\]" | tail -20

# PM2
pm2 logs app-name --lines 50 | grep "\[CAPI\]"

# Verificar warnings de event_id ausente (indica deduplicação quebrada)
grep "deduplicação impossível" /var/log/app.log

# Meta Events Manager: conferência final
# Painel: Events Manager → Pixel → Test Events → verificar "Matched events"
# Meta mostra se evento foi deduplicado com sucesso via ✓ "Deduplicated"
```

---

## 14. Google Ads Conversion Tracking

```javascript
// gtag.js: evento de conversão no momento da confirmação
// Configurar a conversão em Google Ads → Ferramentas → Conversões
gtag('event', 'conversion', {
  send_to: 'AW-XXXXXXXXX/LABEL_DA_CONVERSAO', // AW-ID/label
  value: 297.00,
  currency: 'BRL',
  transaction_id: 'ORDER-123', // evita contar a mesma conversão 2x
});

// Para Lead (sem valor monetário definido):
gtag('event', 'conversion', {
  send_to: 'AW-XXXXXXXXX/LABEL_LEAD',
  // sem value/currency em leads sem valor definido
});
```

**Enhanced Conversions (Google Ads):** envia PII hasheada (email, telefone) para melhorar taxa de match, similar ao CAPI da Meta.

```javascript
gtag('set', 'user_data', {
  email: sha256(email.toLowerCase().trim()), // pré-hashado SHA-256
  phone_number: sha256(phone.replace(/\D/g, '')),
});
// chamar ANTES do evento 'conversion'
```

## 15. TikTok Events API

TikTok Pixel + Events API seguem o mesmo padrão de deduplicação da Meta: `event_id` idêntico nos dois lados.

```typescript
// Server-side: TikTok Events API
const sendTikTokCAPI = async (eventData: {
  event_id: string;
  event_name: string;  // 'SubmitForm', 'Contact', 'Purchase', 'ViewContent'
  event_time: number;
  user: {
    email?: string;    // SHA-256 lowercase sem espaços
    phone_number?: string; // SHA-256 com DDI, só dígitos
    external_id?: string;  // SHA-256
    ip?: string;       // raw
    user_agent?: string;   // raw
    ttclid?: string;   // cookie ttclid raw: equivalente ao _fbc
  };
  properties?: {
    currency?: string;
    value?: number;
    content_id?: string;
    content_name?: string;
  };
  page?: { url: string };
}) => {
  const { TIKTOK_PIXEL_CODE, TIKTOK_ACCESS_TOKEN } = process.env;
  if (!TIKTOK_PIXEL_CODE || !TIKTOK_ACCESS_TOKEN) return;

  await fetch('https://business-api.tiktok.com/open_api/v1.3/event/track/', {
    method: 'POST',
    headers: {
      'Access-Token': TIKTOK_ACCESS_TOKEN,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      pixel_code: TIKTOK_PIXEL_CODE,
      test_event_code: process.env.TIKTOK_TEST_EVENT_CODE || undefined,
      events: [{
        event_id:   eventData.event_id,   // mesmo que o pixel
        event:      eventData.event_name,
        event_time: eventData.event_time,
        user:       eventData.user,
        properties: eventData.properties || {},
        page:       eventData.page || {},
      }]
    })
  });
};
```

```javascript
// Browser: TikTok Pixel com event_id para deduplicação
// ttq é inicializado pelo script do TikTok Pixel (similar ao fbq)
ttq.track('SubmitForm', {
  content_name: 'Kit Inicial',
  currency: 'BRL',
  value: 297,
}, { event_id: eventId }); // mesmo UUID enviado ao servidor
```

**Eventos padrão TikTok:**

| Evento TikTok | Equivalente Meta | Quando |
|---|---|---|
| `ViewContent` | `ViewContent` | Página de produto vista |
| `SubmitForm` | `Lead` | Formulário enviado |
| `Contact` | `Contact` | Clique em WhatsApp/chat |
| `AddToCart` | `AddToCart` | Item no carrinho |
| `PlaceAnOrder` | `InitiateCheckout` | Início de checkout |
| `CompletePayment` | `Purchase` | Compra finalizada |

**Cookie ttclid:** equivalente ao `fbclid` da Meta. Persistir via Set-Cookie HTTP (ITP bypass) e enviar no `user.ttclid`.

## 16. Validação de Schema com Zod

Validar antes de enviar é a diferença entre dados limpos e dados que corrompem relatórios.

```typescript
import { z } from 'zod';

// Schema de validação do payload de lead antes de enviar à CAPI
const LeadTrackingSchema = z.object({
  event_id:         z.string().uuid("event_id deve ser UUID v4"),
  event_source_url: z.string().url("URL inválida"),
  user_agent:       z.string().min(1),
  external_id:      z.string().optional(),
  fbp:              z.string().regex(/^fb\.\d\.\d+\.\d+$/, "_fbp inválido").optional(),
  fbc:              z.string().optional(),
  fbclid:           z.string().optional(),
});

// No endpoint Express:
app.post('/api/leads', async (req, res) => {
  const trackingResult = LeadTrackingSchema.safeParse(req.body.tracking);

  if (!trackingResult.success) {
    console.warn('[Tracking] Payload inválido:', trackingResult.error.flatten());
    // NÃO abortar o lead: o lead em si é válido mesmo sem tracking perfeito
    // Apenas logar e prosseguir sem CAPI (dado ruim não vai à Meta)
  } else {
    // enviar à CAPI com dados validados
    sendMetaCapi(buildCAPIPayload(trackingResult.data));
  }
});
```

**Utility de teste de hash SHA-256** (verificar normalização em produção):
```typescript
// No shell do servidor: node -e "..."
// Email teste: sha256("test@example.com") deve produzir:
// "973dfe0d55c1203c74bc54ab71b5f2fa5c6b3e2fa7d2f61...'
const validateHash = (raw: string, field: 'email' | 'phone' | 'name') => {
  const hashed = normalize[field](raw);
  console.log(`${field}("${raw}") = ${hashed}`);
  return hashed;
};
```

## 17. Performance Budget: Impacto de Scripts de Tracking no Core Web Vitals

Scripts de terceiros são a principal causa de LCP alto e TBT elevado.

**Impacto típico por script:**

| Script | Impacto TBT (p75) | Impacto LCP |
|---|---|---|
| Meta Pixel (fbevents.js ~80KB) | +150-300ms | Indireto via parse |
| Google Tag Manager | +80-200ms | Bloqueia se no `<head>` sem `async` |
| GA4 (gtag.js) | +50-150ms | Mínimo se async |
| TikTok Pixel | +100-250ms | Similar ao Meta |

**Regras para minimizar impacto:**

```html
<!-- ✅ CORRETO: async + preconnect -->
<link rel="preconnect" href="https://connect.facebook.net" crossorigin />
<script async src="https://connect.facebook.net/en_US/fbevents.js"></script>

<!-- ✅ CORRETO: GTM no <head> mas sem bloquear -->
<!-- O snippet padrão do GTM já tem async/defer embutido -->

<!-- ❌ ERRADO: script síncrono de terceiro no <head> -->
<script src="https://analytics.terceiro.com/track.js"></script>

<!-- ✅ Para eventos não-críticos: carregar depois do LCP -->
<script>
  // Dispara após LCP usando PerformanceObserver
  new PerformanceObserver((list) => {
    for (const entry of list.getEntries()) {
      if (entry.entryType === 'largest-contentful-paint') {
        loadNonCriticalTracking(); // carrega scripts secundários
      }
    }
  }).observe({ type: 'largest-contentful-paint', buffered: true });
</script>
```

**Prioridade de carregamento:**

```
Crítico (no <head>):     Meta Pixel init, fbq queue, GA4 init
                         → devem estar disponíveis quando o usuário age

Diferido (após LCP):     Hotjar, Clarity, scripts de replay, A/B testing
                         → não afetam conversão diretamente

Lazy (após interação):   Chat widgets, CRM trackers, scripts de suporte
                         → carregar só quando necessário
```

## 18. Monitoramento e Alertas de Tracking

Tracking quebrado em produção pode passar dias sem ser detectado: e nesse período os algoritmos de bid perdem sinal.

**Métricas para monitorar:**

```
Diariamente:
  - Volume de eventos Lead/Purchase (alerta se cair > 30% vs. média 7d)
  - Taxa de correspondência CAPI (alerta se cair < 60%)
  - Warnings de event_id ausente nos logs
  - Erros HTTP da CAPI (não-204/200 nas respostas)

Semanalmente:
  - Taxa de deduplicação no Events Manager (deve ser ~50% se pixel + CAPI)
  - Comparar leads no CRM vs. eventos Lead no Meta
  - Comparar volume GA4 vs. Meta pixel PageViews (tolerância ±15%)
```

**Alertas simples (Cloud Run / qualquer servidor):**

```typescript
// Health check de tracking: endpoint para monitoramento externo
app.get('/api/tracking/health', (req, res) => {
  const issues: string[] = [];

  if (!process.env.META_PIXEL_ID)    issues.push('META_PIXEL_ID não configurado');
  if (!process.env.META_ACCESS_TOKEN) issues.push('META_ACCESS_TOKEN não configurado');
  if (process.env.META_TEST_EVENT_CODE) issues.push('META_TEST_EVENT_CODE ativo em produção!');

  if (issues.length > 0) {
    return res.status(500).json({ status: 'degraded', issues });
  }
  res.json({ status: 'ok', pixel_id: process.env.META_PIXEL_ID?.slice(-4) }); // últimos 4 dígitos
});
```

**Alertar quando taxa de match cair** (Meta Events Manager API):

```typescript
// Consulta periódica (cron diário): taxa de match < 60% = alerta
const checkMatchRate = async () => {
  const res = await fetch(
    `https://graph.facebook.com/v21.0/${META_PIXEL_ID}/stats?fields=event_stats&access_token=${META_ACCESS_TOKEN}`
  );
  const data = await res.json();
  const matchRate = data.event_stats?.find(e => e.event_name === 'Lead')?.match_rate_approx;
  if (matchRate && matchRate < 60) {
    console.error(`[ALERTA] Taxa de correspondência Lead caiu para ${matchRate}%`);
    // enviar alerta via Slack, email, WhatsApp, etc.
  }
};
```

## 19. Variáveis de Ambiente: Template Completo

```bash
# Meta Pixel / CAPI
META_PIXEL_ID=              # ID numérico: DEVE ser idêntico ao fbq('init', ...)
META_ACCESS_TOKEN=          # System User Token (não Page Token)
META_TEST_EVENT_CODE=       # Apenas em staging: VAZIO em produção

# Google Analytics 4
GA4_MEASUREMENT_ID=         # G-XXXXXXXXXX
GA4_API_SECRET=             # Criado em GA4 > Data Streams > Measurement Protocol

# Google Ads
GADS_CONVERSION_ID=         # AW-XXXXXXXXX
GADS_CONVERSION_LABEL=      # label da conversão específica

# TikTok Pixel / Events API
TIKTOK_PIXEL_CODE=          # código numérico do pixel TikTok
TIKTOK_ACCESS_TOKEN=        # token gerado em TikTok Ads > Assets > Events
TIKTOK_TEST_EVENT_CODE=     # Apenas em staging: VAZIO em produção

# Amplitude (opcional)
AMPLITUDE_API_KEY=

# Mixpanel (opcional)
MIXPANEL_PROJECT_TOKEN=

# Filtro de tráfego interno
INTERNAL_IPS=               # IPs separados por vírgula: 192.168.1.1,10.0.0.1
```

## 20. Boas Práticas Universais de Implementação

Independentemente da plataforma ou projeto, siga sempre:

1. **`crypto.randomUUID()`** para todos os `event_id`: nunca `Date.now() + Math.random()`
2. **Pixel dispara APÓS confirmação do servidor** em formulários: garante que pixel e CAPI disparam juntos ou nenhum dispara
3. **Fire-and-forget com fetch** para eventos de clique em links externos (nova aba, página continua viva)
4. **Log todo evento CAPI** com `event_id`, `event_name` e status da resposta: sem log, sem diagnóstico
5. **`value` só em eventos monetários**: não use `value: 0` em Lead ou Contact
6. **`TEST_EVENT_CODE` vazio em produção**: nunca commitar com valor preenchido
7. **PII sempre hashada server-side**: nunca hash no browser (reversível por rainbow table)
8. **Validar payload com Zod** antes de enviar à CAPI: dado inválido silencioso corrompe relatórios
9. **Filtrar bots e IPs internos** antes de qualquer disparo à plataforma de ads
10. **Nunca disparar conversões em `useEffect` sem guard** de ref em React (StrictMode duplica)
11. **Health check `/api/tracking/health`** em todo projeto: facilita detecção rápida de problemas
12. **Monitorar taxa de match semanalmente**: queda abaixo de 60% indica problema de identity
