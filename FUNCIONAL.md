# Agenda Serviços Turísticos — Especificação Funcional Completa

## 1. Identidade
- **Nome oficial**: Agenda Serviços Turísticos
- **Short name**: Agenda ST
- **Marca/Operadora**: Destinos Maragogi (configurável)
- **Localização**: Maragogi-AL
- **Contato**: Rodrigo (82) 99141-4023 (fixo no rodapé do comprovante)

## 2. Stack Técnica
- **Formato**: Single Page Application (SPA) — HTML + CSS + JS inline em único arquivo (`index.html`)
- **Persistência**: `localStorage` (chaves: `destinos_maragogi_bookings`, `destinos_maragogi_routes`, `destinos_maragogi_settings`)
- **PWA**: Service Worker (`sw.js`), Manifest (`manifest.json`), Add to Home Screen
- **Geração de imagem**: `html2canvas` via CDN (carregamento sob demanda)
- **Ícones**: `icon-192.png`, `icon-512.png`, `icon.svg`, `Logo%20Destinos%20Maragogi.png`
- **Deploy**: GitHub Pages (`rocalaca14.github.io/agenda-servicos-turisticos`)

## 3. Arquitetura do Service Worker (`sw.js`)
- **Cache**: `destinos-maragogi-v3`
- **Estratégia de fetch**:
  - Navegação (HTML): network-first — busca da rede, faz cache para offline, fallback para cache
  - Assets estáticos (manifest.json, ícones, logo): cache-first
- **Assets cacheados no install**: manifest.json, icon-192.png, icon-512.png, icon.svg, Logo%20Destinos%20Maragogi.png
- **Versionamento**: `APP_VERSION` no JS (`const APP_VERSION = '5'`). Se `localStorage.appVersion` não bater, desregistra todos SWs, limpa todos caches, redireciona com `?_v=N`. SW registrado como `sw.js?_v=N`.

## 4. Sistema de Agendamentos

### 4.1. Tipos de Passeio (`formState.tipo`)
| Tipo | Descrição |
|------|-----------|
| `buggy` | Passeio de buggy com roteiro, duração, paradas |
| `piscinas` | Piscinas Naturais com embarcação |

### 4.2. Buggy — Campos do Formulário
- **Roteiro**: seleção por cartão (routeGrid) entre rotas pré-definidas
- **Duração**: botões inline por rota (ex: 2h, 2h30, 4h…)
- **Valor**: calculado automaticamente (`calcValue()`)
- **Quantidade de buggys**: `formState.quantity` (1 padrão, privativo)
- **Personalização**: `customStops` (paradas extras) + `customValue` (valor adicional)
- **Desconto**: `formState.discount` (subtraído do total)
- **Sinal/Entrada**: `formState.signal` (botões 0/50/100% + valor personalizado)
- **Embarque**: `hotel` (com campo de qual hotel) ou `ponto_encontro` (Posto Ipiranga)
- **Status**: confirmado / concluído / cancelado

### 4.3. Buggy — Cálculo de Valor
```
total = (preço_do_roteiro × quantidade) + customValue - discount
```
- `calcValue()`: usa `formState.routeId` + `formState.durationLabel` → busca preço no array de durações
- Exibido em `valueDisplay` com detalhamento (quantidade, personalização, desconto)

### 4.4. Piscinas Naturais — Campos do Formulário
- **Nome da Piscina**: `fPoolName` (texto livre)
- **Tipo de Embarcação** (`vesselType`): Lancha (`lancha`), Jangada (`jangada`), Catamarã (`catamara`)
- **Modalidade** (`vesselMode`, apenas lancha/jangada): Privada (`privada`) ou Compartilhada (`compartilhada`) — catamarã é sempre compartilhada
- **Quantidade de Adultos**: `poolAdults`
- **Quantidade de Crianças (meia)**: `poolChildren`
- **Valor Individual (por pessoa)**: `poolIndividualPrice`
- **Ponto de Embarque**: `fPickupDetail` (texto livre)
- **Sinal/Entrada**: mesmo sistema do buggy (0/50/100/personalizado)
- **Status**: confirmado / concluído / cancelado

### 4.5. Piscinas Naturais — Cálculo de Valor
```
total = (adults × preço_individual) + (children × preço_individual × 0.5) - discount
```
- `poolCalcTotal()`: crianças pagam meia (50% do valor individual)
- `updatePoolTotal()`: atualiza display em tempo real no formulário

### 4.6. Campos Compartilhados (ambos os tipos)
- Data (`fDate`), Horário início (`fStart`), Horário fim (`fEnd`)
- Cliente (`fClient`), Contato (`fContact`), Acompanhantes (`fCompanions`)
- Observações (`fNotes`)
- Sinal (`formState.signal`)
- Status (`formState.status`)

## 5. Estrutura de Dados (Booking Object)
```javascript
// Buggy
{
  id: string (gerado automaticamente),
  tipo: 'buggy',
  routeId: string,
  durationLabel: string,
  value: number,
  customValue: number,
  quantity: number,
  discount: number,
  client: string,
  contact: string,
  companions: string,
  date: string (YYYY-MM-DD),
  startTime: string (HH:MM),
  endTime: string (HH:MM),
  signal: number,
  pickup: string ('hotel'|'ponto_encontro'),
  pickupDetail: string,
  status: string ('confirmado'|'concluido'|'cancelado'),
  notes: string,
  customStops: string,
  createdAt: string (ISO)
}

// Piscinas
{
  id: string,
  tipo: 'piscinas',
  poolName: string,
  poolAdults: number,
  poolChildren: number,
  poolIndividualPrice: number,
  vesselType: 'lancha'|'jangada'|'catamara',
  vesselMode: 'privada'|'compartilhada',
  value: number,
  discount: number,
  client: string,
  contact: string,
  companions: string,
  date: string,
  startTime: string,
  endTime: string,
  signal: number,
  pickupDetail: string,
  status: string,
  notes: string,
  createdAt: string (ISO)
}
```

## 6. Rotas / Roteiros

### 6.1. Estrutura
```javascript
{
  id: string,
  name: string,
  stops: string[],
  durations: [{ label: string, hours: number, price: number }]
}
```

### 6.2. Roteiros Default (5)
| ID | Nome | Durações |
|----|------|----------|
| `norte` | Roteiro Norte (Básico) | 2h (R$230), 2h30 (R$250), 4h (R$350) |
| `ponta_a_ponta` | Ponta a Ponta | 5h (R$450), 7h (R$500) |
| `sul` | Roteiro Sul (Básico) | 3h (R$300), 4h (R$350) |
| `norte_sul` | Norte & Sul | 6h (R$500), 8h (R$600) |
| `milagres` | São Miguel dos Milagres | 7-8h (R$650) |

### 6.3. Persistência
- Armazenado em `localStorage` chave `destinos_maragogi_routes` (editável via modal "Gerenciar Roteiros")
- Fallback: `ROUTES_DEFAULTS` (constante hardcoded)

## 7. Calendário

### 7.1. Visão Mensal (default)
- Navegação por mês (◀ ▶)
- Dias com agendamento: badge circular com quantidade
- **Anéis de segmento** (ring segments): cada booking vira um arco SVG no círculo do dia
  - Cores: confirmado `#00606C`, concluído `#019391`, cancelado `#707070`
  - `stroke-width: 9`, `stroke-linecap: round`
  - Segmentos sequenciais sobre 12h (07:00–19:00)
  - Cada booking ocupa `(horas_do_passeio / 12) × circunferência`
- Hoje: número laranja 22px/900
- Dias sem agendamento: sem anéis

### 7.2. Day Popup (bottom sheet ao clicar no dia)
- Lista de bookings do dia com título, horário, cliente, status
- Subtítulo para piscinas: emoji embarcação + modalidade + adultos
- Botão "Ver Dia Completo →"

### 7.3. Day Detail / Timeline Vertical (07:00–19:00)
- Timeline com 24 slots de 30min (720px total)
- Barra de navegação semanal (7 dias)
- Blocos coloridos por status:
  - Confirmado: fundo `#00606C`, borda turquesa
  - Concluído: fundo `#019391`, borda turquesa
  - Cancelado: fundo `#707070`, borda turquesa
- Cada bloco: título, cliente, horário início/término, embarque + info embarcação (piscinas)
- Altura do bloco = `duração_em_horas × 60px` (mín 40px)
- Clique → modal de detalhes

## 8. Modal de Detalhes (`openDetail`)
- Título: nome do cliente
- Se buggy: roteiro, duração, data, horário, contato (link WhatsApp), acompanhantes, embarque, quantidade de buggys, personalização, desconto, valor total, valor pago, restante, status, observações
- Se piscinas: tipo (Piscinas Naturais), nome da piscina, **embarcação + modalidade**, data, horário, adultos, crianças, valor individual, desconto, valor total, valor pago, restante, contato, acompanhantes, ponto de embarque, status, observações
- Ações: Editar, Comprovante, **Ver Dia Completo**, Excluir, Fechar

## 9. Comprovante / Receipt (`generateReceipt`)

### 9.1. Layout
- Fundo branco com borda gradiente 4px (petróleo → turquesa)
- Padrão decorativo SVG (`opacity: .35`)
- Logo circular (90×90px), nome da empresa, tagline
- Divisores com gradiente

### 9.2. Informações (diferenciadas por tipo)
**Buggy:**
- Data, Horário, Cliente, Contato, Acompanhantes
- Embarque, Quantidade de buggys
- Nome do roteiro + paradas (com ícones: `ric('loc')` para pontos, `ric('food')` para almoço)
- Personalização (se houver)
- Roteiro base, Personalização (+), Desconto (-), Valor Total, Valor Pago, Saldo Restante

**Piscinas:**
- Data, Horário, Cliente, Contato, Acompanhantes
- Embarque, Piscina
- **Embarcação + Modalidade** (🚤/⛵/🛥️ + Privada/Compartilhada)
- Adultos × valor, Crianças × meia
- Header "🏝️ Piscinas Naturais"
- Valor Total, Desconto, Valor Pago, Saldo Restante

### 9.3. Rodapé
- `@destinosmaragogi · Maragogi-AL`
- "Contato Rodrigo: (82) 99141-4023"
- Pill de status

### 9.4. Compartilhamento
- Botão Apple-style no final do modal (SVG share icon)
- `navigator.share()` com arquivo JPEG em anexo (quando suportado)
- Fallback: download direto via `<a>` + `URL.createObjectURL(blob)`
- Geração via `html2canvas` (carregado sob demanda do CDN)

## 10. Paleta de Cores (Sattis)
```css
--t: #00606C    /* Petróleo — títulos, botões primários, header */
--t2: #019391   /* Petróleo claro — hover, pills concluído */
--o: #00DFC3    /* Turquesa — FAB, destaques, botões secundários */
--o2: #00BFA8   /* Turquesa escuro — hover */
--bg: #F4FAFA   /* Fundo */
--border: #B7E5E2 /* Bordas */
--white: #FFFFFF
--text: #102A30
--text2: #527178
```

## 11. Navegação

### 11.1. Bottom Nav (3 abas)
| Ícone | View | Descrição |
|-------|------|-----------|
| 📅 Calendário | `calendar` | Visão mensal com anéis |
| ➕ (FAB) | `form` | Novo agendamento (botão flutuante) |
| 📊 Estatísticas | `stats` | Dashboard financeiro |

- Aba "Novo" removida (FAB é a ação primária de adicionar)
- Nav fixa no bottom, `padding-bottom: env(safe-area-inset-bottom,0)`

### 11.2. Sidebar (hambúrguer no header)
- Logo + nome da empresa
- Filtros: Todos, Confirmados, Concluídos, Cancelados
- Links: Personalização, Lista de Agendamentos, Estatísticas
- Placeholder: "🚁 Voos — em breve"

### 11.3. Views
- `agenda` — lista de agendamentos agrupados por data com filtro
- `form` — formulário de criação/edição
- `calendar` — calendário mensal (view default)
- `stats` — dashboard estatístico
- `dayDetail` — timeline vertical do dia

## 12. Filtros
- Sidebar: Todos / Confirmados / Concluídos / Cancelados
- Agenda: barra de filtro horizontal (mesmos 4)
- Agenda: campo de busca por nome do cliente no topo da lista
- Calendário e timeline: filtro lateral (sidebarFilter)
- Estatísticas: filtro por mês

## 13. Backup de Dados
- Botões Exportar/Importar no modal de Configurações
- Export: baixa arquivo `.json` com todos bookings, rotas e configurações
- Import: carrega arquivo `.json` e substitui dados atuais (com confirmação)

## 14. Estatísticas (`renderStats`)
- **Filtro por mês**: seletor no topo para filtrar por mês
- **Cards**: Total, Confirmados, Concluídos, Cancelados
- **Financeiro**: Receita Total, Recebido, A Receber (com cores)
- **Por Roteiro**: cada roteiro + Piscinas Naturais com total de agendamentos e receita
- **Últimos Agendamentos**: últimos 5 (com tipo label)

## 15. Personalização / Settings

| Campo | Chave | Tipo | Padrão |
|-------|-------|------|--------|
| Nome da empresa | `companyName` | string | "Agenda Serviços Turísticos" |
| Subtítulo | `tagline` | string | "Serviços Turísticos - Maragogi-AL" |
| Localização | `location` | string | "Maragogi-AL" |
| Logo | `logoDataUrl` | data URL | vazio (usa `Logo Destinos Maragogi.png`) |
| Zoom do logo | `logoZoom` | number (0.3–3, step 0.1) | 1 |

- Logo em formato redondo em todos os lugares (`.logo-round` com `border-radius: 50%`)
- Preview 120×120px no settings com zoom

## 16. Service Worker (PWA)

### 16.1. Estratégia de Cache
- **Navegação (HTML)**: Network-first → cache para offline
- **Assets**: Cache-first
- **Cache atual**: `destinos-maragogi-v3`

### 16.2. Versionamento Automático
- `const APP_VERSION = '5'` no JS
- No load: se `localStorage.appVersion !== APP_VERSION`:
  1. Desregistra todos service workers
  2. Limpa todos os caches
  3. Redireciona para `?_v={APP_VERSION}`
- SW registrado como `sw.js?_v={APP_VERSION}` (força novo SW a cada versão)

## 17. FAB (Floating Action Button)
- 56×56px, laranja turquesa (`--o`)
- Ícone SVG `+` centralizado
- Posição: `bottom: 72px`, `left: 50%`, `transform: translateX(-50%)`
- `box-shadow: 0 6px 20px rgba(0,223,195,.45)`

## 18. Modal System
- `openModal(id)` / `closeModal(id)` — toggles classe `.active`
- Fechamento ao clicar no backdrop (overlay)
- Transição `slideUp` (`transform: translateY(60px) → translateY(0)`)
- Máx 90vh, scroll interno

## 19. Gerenciamento de Rotas
- Modal "Gerenciar Roteiros" (`routeEditorModal`)
- Acessível via sidebar > Personalização > (navegação)
- Edição inline dos roteiros, paradas e durações
- Persistido em `localStorage`

## 20. Fluxo de Edição
1. Clique em booking → modal de detalhes → "Editar"
2. Formulário preenchido com dados existentes
3. `editingId` armazena o ID do booking em edição
4. Ao salvar: `bookings[idx] = booking` (substitui)
5. `editingId = null` após salvar

## 21. Inicialização (Init)
1. `DOMContentLoaded` → esconde loading, mostra header
2. `loadSettings()` — carrega personalização do localStorage
3. `loadRoutes()` — carrega rotas do localStorage (fallback default)
4. `loadBookings()` — carrega bookings, sanitiza dados
5. Migra dados antigos (unifica campos, remove obsoletos)
6. `setupNav()` — attach eventos de navegação
7. `renderCalendar()` — renderiza calendário
8. `applySettings()` — aplica nome/logo/tagline

## 22. Arquivos do Projeto
```
/ (raiz)
├── index.html          — App completo (~2260 linhas, HTML+CSS+JS inline)
├── sw.js               — Service Worker (network-first + cache-first)
├── manifest.json       — PWA manifest
├── icon-192.png        — PWA icon 192×192
├── icon-512.png        — PWA icon 512×512
├── icon.svg            — Fallback SVG icon
├── Logo Destinos Maragogi.png — Logo padrão
├── pattern-bg.svg      — (não utilizado)
└── FUNCIONAL.md        — Este documento
```

## 23. Funcionalidades Não Implementadas / Pendentes
- Sistema de notificações (popup 2h antes do passeio + Notification API)
- Recorrência de agendamentos
- Integração com API de pagamentos
- "🚁 Voos — em breve" (placeholder na sidebar)
