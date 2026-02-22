# GaviraLabs — Revisión técnica (arquitectura, calidad, seguridad y escalabilidad)

## 1) Arquitectura actual (resumen)

- Sitio **estático multipágina** (MPA) con HTML plano: `index.html`, guía SEO y páginas legales/contacto.
- Estilo y comportamiento apoyados en CDN externos (Tailwind runtime, GSAP, Highlight.js).
- Configuración de headers de seguridad en `vercel.json`.
- SEO base con `sitemap.xml`, `robots.txt`, metadatos OG y canonical.

### Diagnóstico de arquitectura

La arquitectura es válida para una web corporativa ligera, pero hoy hay señales de crecimiento que piden una estructura más modular:

1. **Duplicación de layout y JS de idioma** en múltiples páginas legales.
2. Dependencia de CDN en tiempo de ejecución para CSS/JS.
3. Ausencia de pipeline de build/lint/test.
4. Sin capa reusable de componentes o plantillas.

---

## 2) Hallazgos por severidad

## Crítico

### C1. Falta de Content Security Policy estricta y HSTS

**Riesgo:** sin CSP/HSTS fuerte, se amplía superficie XSS y downgrade a HTTP.

**Estado recomendado (aplicado):**
- Añadir `Strict-Transport-Security`.
- Añadir `Content-Security-Policy` con `default-src 'self'`, `frame-ancestors 'none'`, `base-uri 'self'`, etc.

**Implementación sugerida (ya incorporada):** en `vercel.json`.

---

## Importante

### I1. Duplicación de `<meta name="robots">` en páginas legales/contacto

**Impacto:** ruido técnico, mantenimiento peor y riesgo de inconsistencias SEO.

**Fix aplicado:** se eliminó el duplicado dejando una sola definición por página.

### I2. Dependencia de Tailwind CDN runtime en todas las páginas

**Impacto:**
- peor TTI/LCP por descargar/ejecutar JS de generación de estilos en cliente,
- menor control de versiones,
- riesgo de disponibilidad de tercero.

**Mejora propuesta:** mover a build local (Tailwind CLI/PostCSS) y servir CSS estático minificado.

### I3. Código duplicado de i18n (setLang + listeners + localStorage)

**Impacto:** coste de cambios elevado (N páginas), posibilidad de bugs divergentes.

**Mejora propuesta:** extraer a `assets/js/i18n.js` compartido y dataset declarativo por página.

```html
<script defer src="/assets/js/i18n.js"></script>
```

```js
// assets/js/i18n.js
export function bootLang({ defaultLang = 'es', storageKey = 'gaviralabs_lang' } = {}) {
  const setLang = (lang) => {
    const isEn = lang === 'en';
    document.documentElement.lang = isEn ? 'en' : 'es';
    document.querySelectorAll('[data-lang]').forEach((el) => {
      el.classList.toggle('hidden', el.dataset.lang !== (isEn ? 'en' : 'es'));
    });
    localStorage.setItem(storageKey, isEn ? 'en' : 'es');
  };
  document.getElementById('lang-es')?.addEventListener('click', () => setLang('es'));
  document.getElementById('lang-en')?.addEventListener('click', () => setLang('en'));
  setLang(localStorage.getItem(storageKey) || defaultLang);
}
```

### I4. `robots.txt` con directiva `Host` no estándar/formato inconsistente

**Impacto:** nulo/ambiguo para buscadores; puede confundir revisiones SEO.

**Fix aplicado:** se dejó `User-agent`, `Allow` y `Sitemap` únicamente.

### I5. Activos de vídeo `hero.mp4`/`hero.webm` a 0 bytes

**Impacto:** señales de deuda técnica / artefactos residuales.

**Mejora propuesta:** eliminar o reemplazar por archivos válidos y referenciarlos explícitamente.

---

## Mejorable

### M1. `X-XSS-Protection` heredado/obsoleto

**Impacto:** header legado con utilidad limitada en navegadores modernos.

**Mejora:** priorizar CSP robusta (ya introducida), pudiendo eliminar completamente ese header.

### M2. Sin estrategia de test/lint automatizada

**Impacto:** regresiones silenciosas en HTML/SEO/links.

**Mejora propuesta:**
- `html-validate` para markup,
- chequeo de enlaces internos,
- Lighthouse CI básico,
- revisión de headers de seguridad en CI.

### M3. Estructura plana de archivos raíz

**Impacto:** escala peor cuando crezcan secciones y assets.

**Mejora propuesta:**

```
/
  pages/
    index.html
    legal/
      privacidad.html
      terminos.html
      cookies.html
      aviso-legal.html
      security.html
    blog/
      crear-anuncios-para-vender-mas.html
  assets/
    css/
    js/
    img/
  docs/
```

---

## 3) Rendimiento: oportunidades concretas

1. **Eliminar Tailwind CDN runtime** y compilar CSS estático.
2. Marcar scripts de terceros como `defer` cuando aplique.
3. Evitar trabajo de `hljs.highlightElement` en cada tick de typing; aplicar highlight por línea o al final del frame (throttle).
4. Revisar payload y cargar GSAP solo en páginas que realmente animan.

Snippet de mejora para typing + highlight más barato:

```js
let rafPending = false;
function paintCode(nextText) {
  codeEl.textContent = nextText;
  if (!rafPending) {
    rafPending = true;
    requestAnimationFrame(() => {
      hljs.highlightElement(codeEl);
      rafPending = false;
    });
  }
}
```

---

## 4) Seguridad: checklist recomendado

- CSP y HSTS (aplicado en configuración).
- Añadir `Cross-Origin-Opener-Policy` y `Cross-Origin-Resource-Policy` según necesidades.
- Añadir SRI (`integrity`) a dependencias CDN o, idealmente, servir librerías localmente.
- Mantener inventario de terceros (Tailwind CDN, jsdelivr).

---

## 5) Escalabilidad y refactor estructural recomendado

### Opción A (mínima fricción)
Mantener HTML estático, pero introducir:
- partials/layout base con un generador estático simple (11ty/Astro/Handlebars prebuild),
- assets versionados,
- build de CSS,
- scripts compartidos.

### Opción B (evolución natural)
Migrar a **Astro o Next.js (SSG)**:
- componentes reutilizables,
- i18n centralizado,
- SEO por plantilla,
- mejor DX para crecer en número de productos/subdominios.

---

## 6) Código duplicado/innecesario detectado

- Bloques repetidos de cabecera legal + selector de idioma + script `setLang`.
- Meta `robots` repetido en varias páginas (ya corregido).
- Activos de vídeo vacíos no usados.

---

## 7) Plan de acción priorizado

1. **Semana 1:** consolidar JS compartido de idioma y layout legal común.
2. **Semana 1:** sustituir Tailwind CDN por build local.
3. **Semana 2:** CI con validaciones (HTML/links/Lighthouse básico/headers).
4. **Semana 2:** limpieza de assets huérfanos y estructura por carpetas.
5. **Semana 3+:** decidir migración a Astro/Next SSG según roadmap de productos.

