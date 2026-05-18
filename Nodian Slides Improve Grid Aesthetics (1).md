---
name: nodian-slides
description: >
  Cria apresentações HTML tipo slideshow na identidade visual Nodian usando o design system
  nodian-slides.css. Use esta skill sempre que o usuário pedir para criar, editar ou revisar
  apresentações de slides, decks, ou relatórios narrativos no ecossistema Nodian — mesmo que
  não mencione "slides" explicitamente. Também se aplica quando o usuário fornecer dados e
  pedir um "relatório", "apresentação", "sprint" ou "painel" visual.
---

# Nodian Slides — Skill de Apresentações

## Arquitetura do sistema

Toda apresentação Nodian usa:

1. **`nodian-slides.css`** — CSS externo e agnóstico (design system completo, ~1270 linhas)
2. **`<style>` patch interno** — apenas overrides específicos do projeto (max ~60 linhas)
3. **ECharts 5.5.1** — gráficos com funções `build*()` dedicadas, inline via `<script>` no final do body
4. **Navegação JS inline** — engine de slideshow (teclado, scroll, touch, progress bar)

---

## Template mínimo de HTML

```html
<!doctype html>
<html lang="pt-br">
<head>
<meta charset="utf-8"/>
<meta name="viewport" content="width=device-width, initial-scale=1"/>
<title>[Título] — Nodian</title>
<link rel="preconnect" href="https://fonts.googleapis.com"/>
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin/>
<link href="https://fonts.googleapis.com/css2?family=Poppins:ital,wght@0,300;0,400;0,500;0,600;0,700;0,900;1,400&display=swap" rel="stylesheet"/>
<script src="https://cdn.jsdelivr.net/npm/echarts@5.5.1/dist/echarts.min.js"></script>
<link rel="stylesheet" href="nodian-slides.css"/>
<style>
/* ═══════════════════════════════
   [PROJETO] PROJECT PATCH
   Overrides específicos deste projeto sobre nodian-slides.css
   ═══════════════════════════════ */
:root {
  /* Aliases curtos para JS dos gráficos ECharts */
  --bg:           #f5f5f5;
  --black:        #161616;
  --gray:         #676767;
  --lgray:        #e0e0e0;
  --white:        #ffffff;
  --amber:        #f6b359;
  --coral:        #e07b5a;
  --sand:         #c6b198;
}

/* Adicione APENAS regras que diferem da base */
</style>
</head>
<body>

<!-- Brand Corner — logo preto, funciona sobre qualquer fundo claro ou âmbar -->
<div class="brand-corner"><span class="brand-corner-logo"></span></div>

<!-- Progress Bar -->
<div class="nav-progress" id="nav-progress" style="width:0%"></div>

<!-- Deck -->
<div class="deck" id="deck">

  <!-- SLIDES VÃO AQUI -->

</div>

<!-- Navigation Bar -->
<div class="nav-bar" id="nav-bar">
  <button class="nav-btn" id="nav-prev" onclick="goSlide(-1)" title="Anterior (↑ / PgUp)">↑</button>
  <div class="nav-counter"><span id="nav-current">1</span><br>/<br><span id="nav-total">1</span></div>
  <button class="nav-btn" id="nav-next" onclick="goSlide(1)" title="Próximo (↓ / Espaço)">↓</button>
</div>
<!-- ⚠️ NÃO adicionar nenhum outro elemento de navegação, hint, tooltip,
     floating button, ou overlay com instruções de teclado/scroll/touch.
     A nav-bar acima é o ÚNICO controle de navegação visível. -->

<script>
// ═══════ ECHART BUILDERS ═══════
function buildExemplo(){
  const el = document.getElementById('chart-exemplo');
  if(!el) return;
  const chart = echarts.init(el);
  chart.setOption({
    // ... opções ECharts usando var(--amber) etc via getComputedStyle
  });
}

// ═══════ LAZY CHART INIT ═══════
const chartBuilders = {
  'chart-exemplo': buildExemplo
};
const chartInitialized = new Set();

function ensureChartsForSlide(n){
  const slide = document.querySelector('.slide[data-slide="'+n+'"]');
  if(!slide) return;
  slide.querySelectorAll('[id^="chart-"]').forEach(el=>{
    if(chartInitialized.has(el.id)){
      const inst = echarts.getInstanceByDom(el);
      if(inst) inst.resize();
    } else if(chartBuilders[el.id]){
      chartBuilders[el.id]();
      chartInitialized.add(el.id);
    }
  });
}

function resizeAllCharts(){
  chartInitialized.forEach(id=>{
    const el = document.getElementById(id);
    if(!el) return;
    const inst = echarts.getInstanceByDom(el);
    if(inst) inst.resize();
  });
}

// ═══════ SLIDESHOW ENGINE ═══════
let currentSlide = 1, totalSlides = 1;

document.addEventListener('DOMContentLoaded', ()=>{
  initDeck();
});

window.addEventListener('resize', ()=>resizeAllCharts());

function initDeck(){
  const slides = document.querySelectorAll('.slide');
  totalSlides = slides.length;
  document.getElementById('nav-total').textContent = totalSlides;
  showSlide(1);

  document.addEventListener('keydown', (e)=>{
    if(e.target.matches('input,textarea')) return;
    switch(e.key){
      case 'ArrowRight': case 'ArrowDown': case 'PageDown': case ' ':
        e.preventDefault(); goSlide(1); break;
      case 'ArrowLeft': case 'ArrowUp': case 'PageUp':
        e.preventDefault(); goSlide(-1); break;
      case 'Home': e.preventDefault(); showSlide(1); break;
      case 'End': e.preventDefault(); showSlide(totalSlides); break;
    }
  });

  let wheelLock = false;
  document.addEventListener('wheel', (e)=>{
    if(wheelLock || Math.abs(e.deltaY) < 30) return;
    wheelLock = true;
    goSlide(e.deltaY > 0 ? 1 : -1);
    setTimeout(()=>{ wheelLock = false; }, 700);
  }, {passive:true});

  let touchY = null;
  document.addEventListener('touchstart', (e)=>{ touchY = e.touches[0].clientY; }, {passive:true});
  document.addEventListener('touchend', (e)=>{
    if(touchY === null) return;
    const dy = touchY - e.changedTouches[0].clientY;
    if(Math.abs(dy) > 50) goSlide(dy > 0 ? 1 : -1);
    touchY = null;
  }, {passive:true});
}

function goSlide(delta){ showSlide(currentSlide + delta); }

function showSlide(n){
  if(n < 1 || n > totalSlides) return;
  currentSlide = n;
  document.querySelectorAll('.slide').forEach(s=>{
    s.classList.toggle('active', +s.dataset.slide === n);
  });
  document.getElementById('nav-current').textContent = n;
  document.getElementById('nav-progress').style.width = ((n-1)/(totalSlides-1)*100) + '%';
  document.getElementById('nav-prev').disabled = (n === 1);
  document.getElementById('nav-next').disabled = (n === totalSlides);

  // ── v1.1 — Capa âmbar: toggle body.amber-slide (NÃO dark-slide) ──
  // Quando o slide ativo é .slide-cover-dark, aplica body.amber-slide
  // para adaptar nav bar (branca semitransparente) e progress bar (preta).
  // Logo da brand-corner permanece preto — NÃO aplicar filter:invert.
  const activeSlide = document.querySelector('.slide.active');
  const isAmber = activeSlide?.classList.contains('slide-cover-dark');
  document.body.classList.toggle('amber-slide', isAmber);

  setTimeout(()=>ensureChartsForSlide(n), 80);
}
</script>
</body>
</html>
```

---

## Anatomia de um deck — ordem dos slides

Um deck Nodian segue esta estrutura narrativa:

```
1. CAPA — slide-cover-dark (⭐ PADRÃO — fundo âmbar, texto preto)
2. ÍNDICE (slide-cover slide-index) — opcional, recomendado para decks com 3+ blocos
3. DIVIDER — bloco temático (slide-cover slide-divider, amber bg)
4. CONTEÚDO — slides de dados/análise (2 a 5 slides por bloco)
5. DIVIDER — próximo bloco
6. CONTEÚDO — mais slides
7. ... repete blocos conforme necessário
8. CLOSING (slide-closing) — síntese final com chaves + cuidados
```

### Numeração dos slides

**NÃO inclua `.slide-num`** nos slides de conteúdo. O contexto de bloco fica implícito no `.slide-eyebrow`.

### Goldilocks Rule (regra dos cachinhos dourados)

Cada slide de conteúdo deve ter **exatamente 1 componente visual principal**:
- 1 gráfico full-width, OU
- 1 split-row (gráfico + insights laterais), OU
- 1 grid de KPIs, OU
- 1 insight grid (horizontal ou vertical), OU
- 1 tabela

Nunca:
- 2 gráficos independentes no mesmo slide
- 0 elementos visuais (slide "só texto")
- Mais de 5 insights/KPIs/steps num componente

---

## Catálogo de Slide Types

### ⭐ Cover Amber — CAPA PADRÃO (`slide-cover-dark`)

> **Esta é a capa padrão de todos os decks Nodian.** Use `slide-cover-dark` como slide 1 em qualquer novo projeto. A classe chama-se `slide-cover-dark` por compatibilidade histórica, mas o fundo é **âmbar** desde a v1.1 — não preto.

Fundo âmbar (`--color-amber`), texto preto (`--color-black`, contraste ~7:1 WCAG AA ✓). Layout alinhado à esquerda, distribuído verticalmente: eyebrow no topo, título + sub centrados, meta-items no fundo. Dois logos decorativos Nodian lado a lado à direita gerados automaticamente pelo CSS `::before` via `repeat-x` (strokes pretos, opacity 0.14) — **nenhum HTML necessário para o fundo decorativo**.

```html
<section class="slide slide-cover-dark" data-slide="1">
  <div class="cover-content">
    <div class="cover-eyebrow">Painel Oncológico · CID-10 C34 · Brônquios e Pulmão</div>
    <h1 class="cover-title">A jornada do paciente com <em>câncer de pulmão</em> no SUS<br>136.261 retratos</h1>
    <p class="cover-sub">Quem chega, em que estádio, em quanto tempo, com que tratamento, em qual estado — e os limites de uma base administrativa para responder essas perguntas.</p>
    <div class="cover-meta">
      <div class="meta-item"><div class="meta-item-lbl">Período</div><div class="meta-item-val">2013 – 2025</div></div>
      <div class="meta-item"><div class="meta-item-lbl">Casos únicos</div><div class="meta-item-val">136.261</div></div>
      <div class="meta-item"><div class="meta-item-lbl">Fonte</div><div class="meta-item-val">PAINEL-Oncologia</div></div>
      <div class="meta-item"><div class="meta-item-lbl">CID-10</div><div class="meta-item-val">C34</div></div>
    </div>
  </div>
</section>
```

**Regras da capa padrão (v1.1):**
- Classe: `slide slide-cover-dark` — **NÃO** adicionar `slide-cover` junto
- Fundo: `var(--color-amber)` — âmbar, **não preto**
- Todos os textos em `var(--color-black)` — contraste ~7,1:1 WCAG AA ✓
- `<em>` no título = itálico em `rgba(22,22,22,.72)` — hierarquia sutil, contraste ~5,4:1 ✓
- `::before` exibe **2 logos Nodian** lado a lado à direita (`repeat-x right center / 50% 100%`) com strokes `#161616` e opacity 0.14 — **100% automático via CSS, zero HTML necessário**
- **Sem bolinha âmbar no eyebrow** — `::before` do eyebrow é `display:none` (fundo já é âmbar)
- **Sem logo/wordmark no eyebrow** — apenas texto de contexto (fonte, CID, assunto)
- **Logo brand-corner**: permanece preto — **NÃO aplicar `filter:invert()`**
- **JS obrigatório**: `showSlide()` deve fazer `body.classList.toggle('amber-slide', isAmber)` quando o slide ativo tem `.slide-cover-dark` — adapta nav bar e progress bar para o fundo âmbar

**Comportamento da nav no slide âmbar (`body.amber-slide`):**
- Nav bar: `rgba(255,255,255,.50)` + `backdrop-filter: blur(10px)` — flutua discretamente
- Botões: `rgba(22,22,22,.06)` com hover `rgba(22,22,22,.14)` — texto preto
- Progress bar: `var(--color-black)` — âmbar sobre âmbar seria invisível

**JS — toggle `amber-slide` (obrigatório em `showSlide`):**
```js
// dentro de showSlide(n):
const activeSlide = document.querySelector('.slide.active');
const isAmber = activeSlide?.classList.contains('slide-cover-dark');
document.body.classList.toggle('amber-slide', isAmber);
// ⚠️ NÃO usar dark-slide — esse seletor foi aposentado na v1.1
```

**⚠️ PROIBIÇÕES DA CAPA PADRÃO — erros que o agente NUNCA deve cometer:**
- ❌ **NUNCA usar fundo preto na capa** — `.slide-cover-dark` tem `background: var(--color-amber)` desde a v1.1
- ❌ **NUNCA usar `body.dark-slide`** — seletor aposentado; o correto é `body.amber-slide`
- ❌ **NUNCA aplicar `filter:invert(1) brightness(10)` no brand-corner-logo** — o logo já é preto e funciona sobre âmbar sem filtro
- ❌ **NUNCA usar `var(--color-amber)` como cor de texto** na capa — razão de contraste 2,1:1, reprovado em WCAG AA
- ❌ **NUNCA usar `color: var(--color-bg)` ou `color: white` nos textos da capa** — fundo é âmbar, texto deve ser preto
- ❌ **NUNCA adicionar `<img>`, `<svg>`, `<div class="cover-logo">` ou qualquer elemento de logo/imagem no HTML** — o fundo decorativo é gerado automaticamente pelo CSS `::before`
- ❌ **NUNCA colocar "Nodian", "NODIAN", wordmark ou nome da marca no `.cover-eyebrow`** — o eyebrow contém APENAS contexto do relatório
- ❌ **NUNCA criar classes como `.cover-logo`, `.nodian-logo-dark`, `.logo-bg-dark`** — não existem no design system

---

### Cover Light (`slide-cover`)

Variante alternativa de capa — fundo claro (`--color-bg`), texto centralizado, dois círculos decorativos concêntricos. Use quando o projeto tem identidade visual específica que conflita com o fundo âmbar, ou quando o cliente solicitar explicitamente. **Na ausência de instrução, sempre usar `slide-cover-dark` (capa âmbar).**

```html
<section class="slide slide-cover" data-slide="1">
  <div class="cover-content">
    <div class="cover-eyebrow">Painel Oncológico · CID-10 C34 · Brônquios e Pulmão</div>
    <h1 class="cover-title">A jornada do paciente com <em>câncer de pulmão</em> no SUS<br>136.261 retratos</h1>
    <p class="cover-sub">Quem chega, em que estádio, em quanto tempo, com que tratamento, em qual estado — e os limites de uma base administrativa para responder essas perguntas.</p>
    <div class="cover-meta">
      <div><div class="meta-item-lbl">Período</div><div class="meta-item-val">2013 – 2025</div></div>
      <div><div class="meta-item-lbl">Casos únicos</div><div class="meta-item-val">136.261</div></div>
      <div><div class="meta-item-lbl">Fonte</div><div class="meta-item-val">PAINEL-Oncologia</div></div>
      <div><div class="meta-item-lbl">CID-10</div><div class="meta-item-val">C34</div></div>
    </div>
  </div>
</section>
```

**Regras do cover light:**
- `<em>` no título = itálico + cor âmbar (decorativo, funciona pois heading grande >= 24px bold)
- `<br>` para forçar quebra de linha no ponto narrativo ideal
- `cover-meta` com 4 items: Período, Volume principal, Fonte, CID/classificação
- Cover usa pseudo-elements `::before`/`::after` para círculos decorativos (automático via CSS)
- **Título narrativo** — nunca descritivo. Ex: "A jornada do paciente", não "Dados de câncer de pulmão"
- **Não requer toggle JS** — fundo claro, nav bar padrão funciona sem adaptação

---

### Index (`slide-cover slide-index`)

```html
<section class="slide slide-cover slide-index" data-slide="2">
  <div class="cover-content cover-content-wide">
    <div class="cover-eyebrow">Roteiro da apresentação</div>
    <h1 class="cover-title">Cinco blocos para responder<br><em>quem, como, quando e onde</em></h1>
    <div class="index-grid">
      <div class="index-card">
        <div class="index-num">00</div>
        <div class="index-name">Fonte de dados</div>
        <div class="index-desc">Como a base é construída, quem ela enxerga e seus limites estruturais</div>
      </div>
      <div class="index-card">
        <div class="index-num">01</div>
        <div class="index-name">Perfil dos pacientes</div>
        <div class="index-desc">Idade, distribuição etária por estádio e onde os pacientes moram</div>
      </div>
      <!-- ... mais cards -->
    </div>
  </div>
</section>
```

**Regras do index:**
- Usar `cover-content-wide` para dar mais espaço ao grid
- `index-num` começa em "00" (base zero) — mesma numeração dos blocos
- `index-desc` = 1 frase curta que antecipa o conteúdo
- Grid responsivo por padrão; override no patch se precisar de colunas fixas: `.index-grid { grid-template-columns: repeat(5, 1fr); }`
- Título do index é narrativo — anuncia a estrutura, ex: "Cinco blocos para responder quem, como, quando e onde"

### Divider (`slide-cover slide-divider`)

```html
<section class="slide slide-cover slide-divider" data-slide="3">
  <div class="cover-content">
    <div class="cover-eyebrow">Bloco 00 de 04</div>
    <h1 class="cover-title">Fonte de dados</h1>
  </div>
</section>
```

**Regras do divider:**
- Classe composta: `slide slide-cover slide-divider`
- Fundo amber (`--color-amber`) com `logo-bg.svg` em baixa opacidade (automático via CSS)
- Texto SEMPRE em preto (`--color-black`) — contraste garantido
- Eyebrow: "Bloco XX de YY"
- Título: nome do bloco (curto, 2–4 palavras)
- **NÃO requer `divider-list`** — a versão mínima (eyebrow + título) é o padrão
- `divider-list` é OPCIONAL — usar apenas quando o bloco tem sub-temas que vale anunciar

### Slide de conteúdo (padrão)

```html
<section class="slide" data-slide="4">
  <div class="slide-inner">
    <div class="slide-head">
      <div>
        <div class="slide-eyebrow">Construção do dado</div>
        <h2 class="slide-title">Cinco sistemas, uma chave composta: <span style="color:var(--amber)">CID + CNS master</span> amarra o caminho</h2>
      </div>
    </div>
    <p class="slide-lead">Parágrafo de contexto com <strong>destaques em negrito</strong> para os números e conceitos-chave.</p>
    <div class="slide-body">
      <!-- componente visual principal -->
    </div>
  </div>
</section>
```

**Regras do slide de conteúdo:**
- `.slide-eyebrow` = tema/subtema em maiúsculas (automático via CSS)
- `.slide-title` = NARRATIVO, comunica o achado principal. Pode usar `<span style="color:var(--amber)">` para destacar um trecho
- `.slide-lead` = 1–3 frases de contexto interpretativo. Usa `<strong>` para números/conceitos-chave
- `.slide-body` = contém exatamente 1 componente visual (ver Goldilocks)

### Closing (`slide-closing`)

```html
<section class="slide slide-closing" data-slide="28">
  <div class="closing-content">
    <h2 class="closing-headline">O retrato — e três cuidados ao usar esses números</h2>
    <div class="closing-summary">
      <div class="closing-row">
        <div class="closing-key">Base</div>
        <div class="closing-val">PAINEL-Oncologia · CID-10 C34 · 136.261 pacientes únicos · 2013–2025</div>
      </div>
      <div class="closing-row">
        <div class="closing-key">Quem</div>
        <div class="closing-val">54% masculino · mediana 64 anos · 87% com 50 anos ou mais</div>
      </div>
      <div class="closing-row">
        <div class="closing-key">Onde</div>
        <div class="closing-val">Sudeste 42% · Sul 29% · 3% migra entre UFs</div>
      </div>
      <div class="closing-row">
        <div class="closing-key">Como chegam</div>
        <div class="closing-val">62% em E3+E4 · cirurgia em só 12% · QT em 56%</div>
      </div>
      <div class="closing-row">
        <div class="closing-key">Quando</div>
        <div class="closing-val">Mediana 37d até tratamento · 54% cumpre lei 60d</div>
      </div>
    </div>
    <div class="closing-cuidados">
      <div class="cuidado-card">
        <div class="cuidado-num">CUIDADO 01</div>
        <div class="cuidado-txt"><strong>Título do cuidado.</strong> Explicação breve do porquê.</div>
      </div>
      <div class="cuidado-card">
        <div class="cuidado-num">CUIDADO 02</div>
        <div class="cuidado-txt"><strong>Título do cuidado.</strong> Explicação breve.</div>
      </div>
      <div class="cuidado-card">
        <div class="cuidado-num">CUIDADO 03</div>
        <div class="cuidado-txt"><strong>Título do cuidado.</strong> Explicação breve.</div>
      </div>
    </div>
  </div>
</section>
```

**Regras do closing:**
- Fundo claro (`--color-bg`) — NUNCA preto
- `.closing-headline` = narrativo, pode incluir "e X cuidados ao usar esses números"
- `.closing-summary` = 4–6 rows tipo chave-valor sintetizando os achados do deck
- `.closing-key` = 1–2 palavras (Base, Quem, Onde, Como, Quando). Usa `var(--txt-amb)` — NUNCA `var(--color-amber)`
- `.closing-val` = dados separados por ` · ` (middle dot) — formato compacto telegráfico
- `.closing-cuidados` = 3 cards com ressalvas/limitações metodológicas (SEMPRE incluir)
- Os cuidados usam `<strong>` para o conceito-chave seguido de explicação

---

## Catálogo de Content Components

### KPI Strip

```html
<div class="kpi-strip">
  <div class="kpi kpi-amber"><div class="kpi-accent"></div>
    <div class="kpi-n">136.261</div>
    <div class="kpi-lbl">Pacientes únicos C34</div>
    <div class="kpi-sub">1 linha = 1 CNS</div>
  </div>
  <div class="kpi"><div class="kpi-accent"></div>
    <div class="kpi-n">12<small>anos</small></div>
    <div class="kpi-lbl">Cobertura temporal</div>
    <div class="kpi-sub">jan/2013 – abr/2025</div>
  </div>
</div>
```

- Variantes de cor: `kpi-pink`, `kpi-blue`, `kpi-amber`, `kpi-green`, `kpi-lilac`, `kpi-coral`, `kpi-masc`, `kpi-fem`
- `<small>` dentro de `.kpi-n` para unidade/sufixo
- `.kpi-sub` = contexto em 1 linha (período, critério, ou comparação)
- Max 4 KPIs por strip — layout é grid 2×2 (`grid-template-columns: repeat(2, 1fr)`) — com 4 cards ficam 2 em cima e 2 embaixo
- Dentro de `.slide`, KPI strip usa `flex: none` (altura pelo conteúdo, **não estica** para preencher o slide). Isso já está no CSS base (`nodian-slides.css` linha ~1100: `.slide .kpi-strip { flex: none; }`). **NÃO adicione `flex:1`, `height:100%`, ou `align-self:stretch` no `.kpi-strip` — isso quebra o hug.**

### KPI Strip com Ícone Temático

Variante do KPI Strip onde cada card exibe um ícone SVG no topo, substituindo o ponto `.kpi-accent`.

**Quando usar:** slides de overview onde cada KPI representa uma dimensão visual distinta (ex: dados demográficos, causas de óbito, cobertura temporal). Não usar quando os 4 KPIs são da mesma categoria (ex: 4 faixas etárias).

**Quando NÃO usar:**
- ❌ Em slides com layout `split-row` (KPIs ao lado de gráfico) — usar KPI simples sem ícone
- ❌ Quando os KPIs são da mesma categoria (ex: 4 faixas etárias)
- Regra geral: **ícones só aparecem quando o KPI strip é o componente principal do slide (full-width), nunca em sidebar ou split**

**CSS obrigatório no `<style>` patch do projeto:**

```css
/* KPI com ícone temático */
.kpi.kpi-icon .kpi-accent { display: none; }
.kpi-icon-wrap { display: block; margin-bottom: 20px; }
.kpi-icon-wrap svg { display: block; }
```

**HTML — estrutura:**

```html
<div class="kpi-strip">
  <div class="kpi kpi-amber kpi-icon">
    <div class="kpi-accent"></div>
    <div class="kpi-icon-wrap">
      <svg width="28" height="28" viewBox="0 0 28 28" fill="none">
        <!-- SVG temático aqui -->
      </svg>
    </div>
    <div class="kpi-n">17.359</div>
    <div class="kpi-lbl">Óbitos totais com C91.1</div>
    <div class="kpi-sub">1996–2023 · 28 anos de série</div>
  </div>
</div>
```

**Regras do SVG:**
- Tamanho fixo: `width="28" height="28" viewBox="0 0 28 28"`
- Sem `<defs>`, `<filter>`, `<mask>`, `<text>`, `<clipPath>`
- `stroke-width` entre 1 e 1.5
- Paleta restrita: cor principal da variante + seu fill 25%
- Ícone legível em 28px sem legenda

**Mapeamento de cores por variante:**

| Variante   | Stroke    | Fill      |
|------------|-----------|-----------|
| kpi-amber  | `#f6b359` | `#ffe5c7` |
| kpi-blue   | `#7295cd` | `#dbe4f3` |
| kpi-green  | `#55b583` | `#d4ecdf` |
| kpi-lilac  | `#ab87bd` | `#e6dcec` |
| kpi-coral  | `#e07b5a` | `#f8dcd0` |
| (sem var.) | `#676767` | `#e0e0e0` |

**Erros a evitar:**
- ❌ Omitir o `<div class="kpi-accent"></div>` — o CSS precisa do elemento para aplicar `display:none`
- ❌ Usar `<text>` dentro do SVG — não renderiza consistentemente
- ❌ SVG sem `fill="none"` no elemento raiz — herda fills indesejados
- ❌ Misturar cards com e sem ícone no mesmo `.kpi-strip`
- ❌ Esquecer o CSS patch no `<style>` do projeto

### Chart Wrap

```html
<div class="chart-wrap">
  <div class="chart-head">
    <span class="chart-title">Casos por faixa etária IBGE quinquenal — 18 grupos completos</span>
    <span class="chart-src">PAINEL-Oncologia · C34 · n=136.261</span>
  </div>
  <div id="chart-age" class="chart-box"></div>
  <div class="footnote">Faixas IBGE quinquenais; <strong>17. 85+ anos</strong> agrupa todas as idades acima.</div>
</div>
```

- `.chart-box.tall` = 460px; `.chart-box.xtall` = 540px
- Dentro de `.slide`, chart-box usa `flex:1; height:auto` (preenche espaço disponível)
- ⚠️ **NUNCA adicione `style="height:...px"` ou `style="flex:none"` inline no `.chart-box` dentro de slides.** O CSS já cuida do dimensionamento via `flex:1; height:auto !important`. Qualquer inline style de height ou flex no chart-box **impede o gráfico de preencher a altura do slide**. O chart-box deve ser SEMPRE `<div id="..." class="chart-box"></div>` sem inline styles de dimensão.
- **chart-title**: descritivo-técnico (o que o gráfico mostra). O título NARRATIVO vai no `.slide-title`
- **chart-src**: FONTE · CID · n=XXXXX (formato padrão)
- **footnote** dentro do chart-wrap: notas metodológicas sobre aquele gráfico específico
- Use `<strong>` na footnote para destacar valores e categorias mencionadas

### Split Row (gráfico + KPIs ou insights laterais)

```html
<div class="split-row" style="grid-template-columns: 200px 1fr; gap:20px;">
  <div class="split-side" style="justify-content:center; gap:14px;">
    <div class="kpi kpi-amber"><div class="kpi-accent"></div>
      <div class="kpi-n">136.261</div>
      <div class="kpi-lbl">Total de pacientes</div>
      <div class="kpi-sub">CID-10 C34 · 2013–2025</div>
    </div>
    <!-- mais KPIs empilhados -->
  </div>
  <div class="chart-wrap">
    <div class="chart-head">
      <span class="chart-title">% masculino vs. feminino por faixa etária</span>
      <span class="chart-src">PAINEL-Oncologia · C34 · n=136.261</span>
    </div>
    <div id="chart-sexo-faixa" class="chart-box"></div>
    <div class="footnote">Nota explicativa.</div>
  </div>
</div>
```

- Proporções via inline style quando diferem do padrão (200px sidebar + 1fr main)
- `.split-row.equal` = duas colunas iguais (1fr 1fr) — sem inline style
- Padrão base (sem override): `1fr 320px` (gráfico à esquerda, insights à direita)

### Insight Grid

O insight grid tem **2 layouts** e **2 semânticas**, combináveis livremente:

**Layouts:**
- **Horizontal** (padrão) — cards lado a lado em grid responsivo
- **Vertical** (`.insight-grid.vertical`) — cards empilhados, dot à esquerda do texto

**Semânticas:**
- **Achado** (padrão) — fundo branco, dot na cor da variante (`amber`, `blue`, `green`, etc.)
- **Alerta** (`.insight.alert`) — fundo coral claro (`--bg-coral`), dot coral, label em `--txt-coral`. Use para limitações, erros, ressalvas

#### Insight Grid — Horizontal (achados)

```html
<div class="insight-grid">
  <div class="insight amber">
    <div class="insight-dot">1</div>
    <div class="insight-label">ACHADO</div>
    <div class="insight-text"><strong>Texto forte</strong> e contexto interpretativo.</div>
  </div>
</div>
```

#### Insight Grid — Vertical (alertas/limitações)

Use vertical para insights sequenciais ou quando cada item tem label+texto longo. A semântica `.alert` é independente — pode-se usar vertical neutro também:

```html
<div class="insight-grid vertical">
  <div class="insight alert">
    <div class="insight-dot">1</div>
    <div class="insight-label">Não é incidência</div>
    <div class="insight-text">O PAINEL captura <strong>casos atendidos no SUS</strong>, não o universo real de pacientes.</div>
  </div>
</div>
```

**Regras do insight grid:**
- `.insight-dot` contém APENAS numeração (1, 2, 3...) — nunca ícones, emojis ou letras
- Max 4 cards por grid — além disso fica com overload cognitivo
- `.insight-grid.vertical` é obrigatório quando há 3+ alertas (limitações metodológicas)

---

## Decisões de projeto fixas

| Decisão | Valor |
|---|---|
| CSS externo | nodian-slides.css (único para todos os projetos) |
| Patch interno | `<style>` com max ~60 linhas de overrides |
| Font | Poppins (300, 400, 500, 600, 700) via Google Fonts |
| Gráficos | ECharts 5.5.1 via CDN, funções `build*()` inline, lazy init via `chartBuilders` |
| Chart init | NUNCA no DOMContentLoaded — via `ensureChartsForSlide(n)` após slide ficar visível |
| Brand corner | `<span class="brand-corner-logo"></span>` (SVG embutido no CSS como data URI) |
| `--max` | 1080px (page) / 1240px (slide-inner) |
| `--radius` | 30px (cards, chart-wrap) / 16px (cards menores) |
| `.slide-num` | NÃO incluir — contexto de bloco fica no eyebrow |
| **Capa padrão** | **`slide-cover-dark` — fundo âmbar (`--color-amber`), texto preto ⭐** |
| Divider bg | `--color-amber` + logo-bg.svg decorativo (embutido no CSS como data URI) |
| Closing bg | `--color-bg` (claro) — NUNCA preto |
| Numeração blocos | Base zero: 00, 01, 02, 03... |
| Nav bar | `onclick="goSlide(-1)"` / `onclick="goSlide(1)"` inline |
| Resize | `window.addEventListener('resize', resizeAllCharts)` + `ensureChartsForSlide(n)` com `setTimeout 80ms` no `showSlide` |
| **Toggle capa** | **`body.amber-slide` (NÃO `dark-slide`) quando `slide-cover-dark` está ativo** |

---

## Sequência de decisão ao criar um deck completo

1. Definir público e Grande Ideia (storytelling.md Passo 0)
2. Listar os blocos temáticos (3–5 blocos é ideal)
3. Para cada bloco, listar os achados (2–5 slides por bloco)
4. **Montar: `slide-cover-dark` (capa âmbar) → Index → [Divider + Slides]×N → Closing**
5. Numerar `data-slide` sequencialmente (1, 2, 3...)
6. Escrever `closing-summary`: 1 row por bloco + "Base" no topo
7. Escrever 3 cuidados/ressalvas metodológicas
8. Implementar chart builders no JS (funções `build*()`)
9. Registrar todos os charts em `chartBuilders = { 'chart-id': buildFn }`
10. Garantir que `showSlide()` faz toggle `body.amber-slide` (não `dark-slide`)

---

## Erros críticos — NUNCA fazer

### Capa
- ❌ Usar `slide-cover` (light) como capa padrão — usar sempre `slide-cover-dark`
- ❌ Usar fundo preto na capa — `slide-cover-dark` é âmbar desde a v1.1
- ❌ Usar `body.dark-slide` — seletor aposentado, substituído por `body.amber-slide`
- ❌ Aplicar `filter:invert()` no brand-corner-logo na capa âmbar — logo já é preto
- ❌ Usar `var(--color-amber)` como cor de texto (ratio 2,1:1 — reprovado WCAG)
- ❌ Usar textos brancos ou claros sobre fundo âmbar — contraste insuficiente

### Gráficos
- ❌ Chamar `build*()` no DOMContentLoaded (slides hidden → canvas 0×0 → gráfico vazio)
- ❌ Usar SVG inline ou trocar renderer como "solução" para gráficos vazios (o problema é timing)
- ❌ Chart builder sem `if(!el) return` guard (crashes se ID não existe)
- ❌ Esquecer de registrar chart em `chartBuilders` (gráfico nunca é inicializado)
- ❌ Adicionar `style="height:...px"` ou `style="flex:none"` inline no `.chart-box`
- ❌ Usar `kpi-icon` sem o CSS patch no `<style>` do projeto

### Estrutura
- ❌ Fundo preto em capas/closings/dividers (identidade Nodian é clara/âmbar)
- ❌ Mais de 1 gráfico independente por slide (quebra Goldilocks)
- ❌ Slide sem `.slide-lead` (falta contexto interpretativo)
- ❌ Chart sem `.footnote` (gráfico sem fonte/nota)
- ❌ Títulos descritivos no `.slide-title` (deve ser narrativo)
- ❌ Duplicar CSS do base no patch (o patch deve ter APENAS diferenças)
- ❌ Esquecer `data-slide="N"` sequencial no `<section>` (quebra navegação)
- ❌ Usar `<div>` em vez de `<section>` para slides (semântica)
- ❌ Grid com mais de 5 KPIs ou insights (overload cognitivo)
- ❌ `.insight-dot` com conteúdo que não seja numeração (1, 2, 3...) — nunca ícones, emojis ou letras
- ❌ Insight grid com mais de 4 cards (overload cognitivo)
- ❌ Incluir `.slide-num` no markup (removido do padrão — usar eyebrow para contexto de bloco)
- ❌ Divider sem eyebrow "Bloco XX de YY" (perde contexto de posição)
- ❌ Closing sem `.closing-cuidados` (sempre incluir 3 ressalvas)

---

## ECharts — Padrão de implementação

### Estrutura de cada gráfico

```javascript
function buildNomeDoGrafico(){
  const el = document.getElementById('chart-nome');
  if(!el) return;
  const chart = echarts.init(el);
  const s = getComputedStyle(document.documentElement);
  const amber = s.getPropertyValue('--amber').trim() || '#f6b359';
  const black = s.getPropertyValue('--black').trim() || '#161616';
  const gray  = s.getPropertyValue('--gray').trim()  || '#676767';
  const lgray = s.getPropertyValue('--lgray').trim() || '#e0e0e0';
  chart.setOption({
    // ... configuração ECharts
  });
}
```

### Lazy init (CRÍTICO — NÃO PULAR)

```javascript
// REGISTRE todos os charts aqui — o engine faz o resto
const chartBuilders = {
  'chart-age':    buildAge,
  'chart-sexo':   buildSexo,
  'chart-volume': buildVolume
};
```

### Estilo ECharts padrão Nodian

```javascript
// Grid
grid: { top: 20, right: 20, bottom: 32, left: 48 }
// Eixos
axisLine:  { lineStyle: { color: '#e0e0e0' } }
axisTick:  { show: false }
axisLabel: { fontSize: 10, color: '#676767', fontFamily: 'Poppins' }
splitLine: { lineStyle: { color: 'rgba(22,22,22,.05)' } }
// Tooltip
tooltip: {
  trigger: 'axis',
  backgroundColor: '#161616',
  borderColor: '#161616',
  textStyle: { color: '#f5f5f5', fontFamily: 'Poppins', fontSize: 12 }
}
// Barras
barMaxWidth: 28
barCategoryGap: '40%'
borderRadius: [4, 4, 0, 0]
// Linhas
lineStyle: { width: 3 }
```

---

## Regras de acessibilidade

### Contraste WCAG AA — Referência rápida

| Cor | Hex | Ratio vs branco | Uso permitido |
|---|---|---|---|
| Preto | #161616 | 19.1:1 | Texto, labels, qualquer uso |
| Cinza | #676767 | 5.7:1 | Texto secundário |
| Vermelho | #e3404a | 4.6:1 | Labels de status (margem estreita) |
| Azul | #4a7bbf | 4.7:1 | Labels de série |
| Verde | #55b583 | 3.0:1 | Só formas grandes — nunca texto |
| Âmbar | #f6b359 | 2.1:1 | Só preenchimento — nunca texto |
| Lilás | #ab87bd | 3.0:1 | Só formas grandes — nunca texto pequeno |
| Coral | #e07b5a | 3.1:1 | Só formas grandes — nunca texto pequeno |
| Sand | #c6b198 | 2.0:1 | Só fundo/área — nunca texto |

### Texto sobre fundo âmbar (capa padrão)

| Elemento | Valor | Ratio | Status |
|---|---|---|---|
| Título, meta-val | `#161616` | ~7.1:1 | ✅ AAA |
| Título `<em>` | `rgba(22,22,22,.72)` | ~5.4:1 | ✅ AA |
| Subtítulo | `rgba(22,22,22,.62)` | ~4.5:1 | ✅ AA |
| Eyebrow, meta-lbl | `rgba(22,22,22,.55)` | ~4.0:1 | ✅ AA (texto grande) |

### Regras de implementação

| Regra | Implementação |
|---|---|
| Âmbar nunca como texto | Use `--txt-amb` (#8a5a00) para texto/labels |
| Coral nunca como texto | Use `--txt-coral` (#a13d20) para texto/labels |
| Verde/vermelho nunca como únicas pistas | Sempre acompanhar com seta (↑↓) ou prefixo (+/−) |
| Âmbar e coral não juntos sem diferenciador | Adicionar forma ou label textual |
| **Texto sobre fundo amber** | **Sempre `--color-black` (#161616)** |
| `.closing-key` | Usa `--txt-amb`, nunca `--color-amber` |
| `.cover-title em` | Decorativo — `rgba(22,22,22,.72)` sobre âmbar (~5.4:1 ✓) |
| `<span style="color:var(--amber)">` em títulos | OK se texto grande (>= 18px bold) sobre fundo claro |

---

## Padrão de escrita (tom narrativo Nodian)

### Títulos de slide (`.slide-title`)

O título comunica o ACHADO, não descreve o visual.

| ✗ Descritivo | ✓ Narrativo |
|---|---|
| "Distribuição por faixa etária" | "Pico em 65–69 anos; 90% dos casos têm 50 anos ou mais" |
| "Modalidade de tratamento" | "QT lidera com 56%; cirurgia em apenas 11,7%" |
| "Tempo até tratamento" | "Mediana de 37 dias — mas E4 trata mais rápido que E1" |
| "Dados por região" | "Sudeste responde por 42%; Sul concentra 29% com 14% da população" |

Padrão: [Achado principal]; [achado secundário ou contraste]

### Títulos de gráfico (`.chart-title`)

Aqui é descritivo-técnico — diz O QUE o gráfico mostra:
- "Casos por faixa etária IBGE quinquenal — 18 grupos completos"
- "% masculino vs. feminino por faixa etária (100% stacked)"
- "Distribuição de modalidades dentro de cada estádio (% empilhada)"

### Slide lead (`.slide-lead`)

1–3 frases que contextualizam o achado. Padrão:
- Abre com o dado/padrão principal
- Usa `<strong>` para números e conceitos-chave (max 3 por parágrafo)
- Fecha com uma ressalva ou implicação

---

## Referências

- `nodian-slides.css` v1.1 — design system completo, logos embutidos como data URI
- `rules.md` — regras semânticas de cor e acessibilidade WCAG
- `storytelling.md` — princípios de storytelling com dados (Cole Nussbaumer Knaflic)
- O logo e o bg do divider estão embutidos como data URI no CSS — nenhum asset externo necessário