# Tareas propuestas tras revisión rápida del repositorio

## 1) Corregir error tipográfico/ortográfico
**Problema detectado:** el nombre del titular aparece como `Alejandro Gavira Garcia` sin tilde en `García` en varias páginas legales y de contacto.

**Tarea propuesta:**
- Reemplazar `Garcia` por `García` en el footer y cualquier bloque legal equivalente.
- Revisar consistencia del nombre en metadatos estructurados (`JSON-LD`) y páginas estáticas.

**Criterio de aceptación:** no quedan ocurrencias de `Alejandro Gavira Garcia` en el sitio y se mantiene el mismo valor en todas las páginas.

---

## 2) Solucionar fallo funcional (selector de idioma)
**Problema detectado:** los scripts de idioma asumen que `localStorage` siempre está disponible (`getItem`/`setItem`) y esto puede lanzar excepciones (modo privado/restricciones del navegador), rompiendo la inicialización del idioma.

**Tarea propuesta:**
- Encapsular el acceso a `localStorage` en utilidades seguras con `try/catch` y fallback en memoria.
- Evitar que un fallo de almacenamiento impida alternar idioma o renderizar contenido.
- Reutilizar el mismo script compartido para no corregir N veces el mismo bug.

**Criterio de aceptación:** al simular fallo de `localStorage`, la página carga y el cambio ES/EN sigue funcionando.

---

## 3) Corregir discrepancia de documentación/comentario
**Problema detectado:** hay comentarios visibles de "NO VIDEO" en páginas principales, pero siguen existiendo `assets/hero.mp4` y `assets/hero.webm` (a 0 bytes), lo que genera deuda técnica y contradicción entre intención y artefactos.

**Tarea propuesta:**
- Elegir una vía: (a) eliminar los assets vacíos si no se usarán, o (b) sustituirlos por archivos reales y documentar su uso.
- Actualizar `docs/TECHNICAL_REVIEW.md` para reflejar el estado final (resuelto o planificado).

**Criterio de aceptación:** no hay contradicción entre comentarios/documentación y el contenido real de `assets/`.

---

## 4) Mejorar cobertura de pruebas
**Problema detectado:** no existe una prueba automatizada para validar rutas críticas de contenido estático (idioma, enlaces internos, metadatos mínimos).

**Tarea propuesta:**
- Añadir una suite mínima (por ejemplo Playwright o script Node) que verifique:
  - Conmutación de idioma ES/EN en al menos `index.html` y `contacto.html`.
  - Ausencia de enlaces internos rotos en páginas listadas en `sitemap.xml`.
  - Presencia de metadatos SEO básicos (`title`, `description`, `canonical`).
- Ejecutar la suite en CI para evitar regresiones.

**Criterio de aceptación:** existe un comando único de test que falla si se rompe cualquiera de los checks anteriores.
