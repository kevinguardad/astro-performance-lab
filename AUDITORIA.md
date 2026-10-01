# Bitácora de Auditoría — AstroBitácora

**Estudiante:** ______________________
**Repositorio:** ______________________
**URL publicada:** ______________________
**Fecha:** ______________________

---

## 1. Medición inicial (línea base)
*Dispositivo / modo:* Móvil · PageSpeed Insights · fecha y hora: __________

| Indicador | Resultado inicial | Observación |
| :--- | :--- | :--- |
| **Desempeño** | __ / 100 | |
| **LCP** | __ s | |
| **CLS** | __ | |
| **INP** | __ ms (o "sin datos de campo") | |
| **Bytes transferidos** | __ MB | |

### Tres hallazgos principales
1. **Hallazgo:** Imágenes de la galería se descargan de inmediato aunque están fuera del primer viewport.
   * **Evidencia:** (pega aquí lo que viste en Network / Lighthouse: cuántas imágenes y cuántos MB)
   * **Recurso o archivo relacionado:** las 5 `<img>` de `.archive` en `index.html`.
2. **Hallazgo:** Imágenes sin `width` y `height` y servidas en JPG a resolución original.
   * **Evidencia:** (diagnóstico "Imágenes sin dimensiones explícitas" / "Sirve imágenes con formatos modernos" + peso del hero)
   * **Recurso o archivo relacionado:** hero y tarjetas en `index.html`.
3. **Hallazgo:** Script cargado en el `<head>` sin `defer`.
   * **Evidencia:** (diagnóstico "Elimina los recursos que bloquean el renderizado", si aparece)
   * **Recurso o archivo relacionado:** `script.js`.

---

## 2. Hipótesis antes de modificar
1. *Si cambio* el hero a WebP/AVIF con ancho acorde (`f_auto,q_auto,w_1600`) y `fetchpriority="high"`, *espero mejorar* el LCP y los bytes *porque* el hero es el elemento más grande y hoy se descarga a resolución original en JPG.
2. *Si cambio* las imágenes de la galería agregando `width`, `height` y `loading="lazy"`, *espero mejorar* el CLS y los bytes iniciales *porque* el navegador reserva el espacio y no descarga lo que está fuera del viewport.
3. *Si cambio* el script agregando `defer`, *espero mejorar* el bloqueo del render *porque* el parser no se detiene a descargar/ejecutar JS (impacto esperado pequeño: el JS pesa muy poco).

---

## 3. Cambios aplicados

| Commit | Cambio | Motivo | Archivo(s) | Resultado esperado |
| :--- | :--- | :--- | :--- | :--- |
| `perf: optimizar imagen hero` | `f_auto,q_auto,w_1600`, `width/height`, `fetchpriority="high"` | Hero pesado y sin dimensiones | `index.html` | Menor LCP y menos bytes |
| `perf: diferir imagenes fuera del viewport` | `loading="lazy"`, `width/height`, formato moderno en tarjetas | Evitar descargas anticipadas y saltos de layout | `index.html` | Menos bytes iniciales, CLS ≈ 0 |
| `perf: evitar bloqueo del script principal` | `defer` en `script.js` | Script en `<head>` bloquea el parser | `index.html` | DOM se construye sin pausas |

---

## 4. Medición final
*Mismas condiciones que la inicial (móvil, misma herramienta, misma red).* Fecha y hora: __________

| Indicador | Antes | Después | Diferencia |
| :--- | :--- | :--- | :--- |
| **Desempeño** | | | |
| **LCP** | | | |
| **CLS** | | | |
| **INP** | | | |
| **Bytes transferidos** | | | |

---

## 5. Conclusión breve
* **Cambio con mayor impacto:**
* **Cambio con menor impacto:**
* **Problema que todavía queda pendiente:**
* **¿Qué harías en una segunda iteración?** (ej. `srcset`/`<picture>` para servir tamaños distintos por dispositivo, minificar `styles.css`)

---

## 6. Respuestas a las preguntas finales
*(Completa los datos con TUS mediciones. La teoría de abajo es la base.)*

1. **LCP y elemento LCP antes/después:** LCP (*Largest Contentful Paint*) es el tiempo hasta que se pinta el elemento de contenido más grande visible. Antes: ________ (elemento y tiempo). Después: ________. (Lighthouse lo muestra en "Elemento de Largest Contentful Paint".)
2. **Imagen sin dimensiones y CLS:** sin `width`/`height` el navegador no sabe cuánto espacio reservar; cuando la imagen termina de descargarse, empuja el contenido de abajo y genera un salto de layout que suma al CLS.
3. **`loading="lazy"` en galería vs. hero:** en la galería evita descargar lo que el usuario aún no ve. En el hero, que está visible al cargar, retrasaría su descarga y empeoraría el LCP; ahí conviene `fetchpriority="high"`.
4. **`defer` vs `async`:** ambos descargan sin bloquear el parser. `async` ejecuta apenas termina de bajar (orden no garantizado); `defer` ejecuta en orden después de parsear el HTML. Elegí `defer` porque `script.js` usa elementos del DOM.
5. **Formato elegido y comparación de peso:** WebP/AVIF vía Cloudinary (`f_auto,q_auto`). Hero antes: ____ KB/MB → después: ____ KB (míralo en la pestaña Network de DevTools, columna Size).
6. **Si sube el puntaje pero el LCP sigue alto:** no está terminada; el puntaje es un promedio ponderado y el LCP es un Core Web Vital por sí solo (objetivo ≤ 2.5 s). Hay que seguir atacando el elemento LCP.
