# Bitácora de Auditoría — AstroBitácora

**Estudiante:** Kevin Guardado
**Repositorio:** https://github.com/kevinguardad/astro-performance-lab
**URL publicada:** https://kevinguardad.github.io/astro-performance-lab/
**Fecha:** 1 de octubre de 2026

---

## 1. Medición inicial (línea base)
*Herramienta:* PageSpeed Insights · *Modo:* Celulares (Moto G Power emulado, red 4G lenta)
*Fecha y hora:* 1 oct 2026, ~5:28 p.m. (prueba 1) y 5:37 p.m. (prueba 2)

| Indicador | Resultado inicial | Observación |
| :--- | :--- | :--- |
| **Desempeño** | 74 (prueba 1) / 75 (prueba 2) | Penalizado casi por completo por el LCP |
| **LCP** | 126.9 s (en ambas pruebas) | Rojo. Cargar ~30 MB de imágenes con red 4G lenta tarda minutos |
| **CLS** | 0 | Sin saltos medibles: las imágenes están debajo del primer viewport |
| **INP** | Sin datos | PageSpeed no tiene datos de usuarios reales para esta URL ("No hay datos") |
| **Bytes transferidos** | 31,105 KiB (~30 MB) | Advertencia "Evita cargas útiles de red de gran tamaño" |

Otros valores: FCP 0.8 s · TBT 0 ms · Speed Index 3.5 s (prueba 1) / 2.2 s (prueba 2).
Captura: `evidencias/inicial.png`

### Tres hallazgos principales
1. **Hallazgo:** Las imágenes se sirven pesadas y sin formato moderno.
   * **Evidencia:** "Mejora la entrega de imágenes — ahorro estimado de 30,510 KiB". Casi todo el peso de la página (31,105 KiB) son imágenes. El hero original pesa solo 382 kB, así que ~31 MB corresponden a las 5 imágenes de la galería (estimado por diferencia).
   * **Recurso o archivo relacionado:** hero y las 5 imágenes de la galería en `index.html` (JPG a resolución original).
2. **Hallazgo:** El script se carga en el `<head>` sin `defer`.
   * **Evidencia:** "Solicitudes de bloqueo de renderización — ahorro estimado de 420 ms".
   * **Recurso o archivo relacionado:** `script.js` / `index.html`.
3. **Hallazgo:** Las `<img>` no declaran `width` ni `height`.
   * **Evidencia:** Diagnóstico "Los elementos de imagen no tienen ningún atributo width ni height explícito".
   * **Recurso o archivo relacionado:** todas las `<img>` de `index.html`.

---

## 2. Hipótesis antes de modificar
1. *Si cambio* el hero a un formato moderno con un ancho acorde (`f_auto,q_auto,w_1600`) y le agrego `fetchpriority="high"`, *espero mejorar* el LCP y los bytes transferidos *porque* el hero es el elemento más grande de la página y hoy se descarga a resolución original en JPG.
2. *Si cambio* las imágenes de la galería agregando `loading="lazy"`, `width` y `height` y formato moderno, *espero mejorar* los bytes iniciales y evitar futuros saltos de layout *porque* el navegador no descarga lo que está fuera del viewport y reserva el espacio de cada imagen.
3. *Si cambio* el script agregando `defer`, *espero reducir* el bloqueo de renderización *porque* el parser ya no se detiene a descargar y ejecutar el JS (impacto esperado pequeño: el archivo pesa menos de 1 kB).

**Revisión de las hipótesis después de medir:** la hipótesis 1 se cumplió solo en parte. El hero sí bajó de peso (382 kB → 111 kB), pero no era el principal problema: casi todo el peso estaba en la galería. La hipótesis 2 fue la que más pesó en el resultado. La hipótesis 3 tuvo el efecto pequeño que se esperaba.

---

## 3. Cambios aplicados

| Commit | Cambio | Motivo | Archivo(s) | Resultado esperado |
| :--- | :--- | :--- | :--- | :--- |
| `perf: optimizar imagen hero` | `f_auto,q_auto,w_1600`, `width="1600" height="900"`, `fetchpriority="high"` | Hero pesado, sin dimensiones y es el elemento LCP | `index.html` | Menor LCP y menos bytes |
| `perf: diferir imagenes fuera del viewport` | `loading="lazy"`, `width/height` y `f_auto,q_auto,w_*` en las 5 tarjetas | Evitar descargas anticipadas y reservar espacio | `index.html` | Menos bytes iniciales, CLS estable |
| `perf: evitar bloqueo del script principal` | `defer` en `script.js` | Script en `<head>` bloquea el parser | `index.html` | DOM se construye sin pausas |

---

## 4. Medición final
*Mismas condiciones que la inicial (PageSpeed, Celulares).* *Fecha y hora:* 1 oct 2026, 6:03 p.m.
Capturas: `evidencias/final.png` y `evidencias/peso-final.png`

| Indicador | Antes | Después | Diferencia |
| :--- | :--- | :--- | :--- |
| **Desempeño** | 74 / 75 | 99 | +24 / +25 puntos |
| **LCP** | 126.9 s | 2.1 s | −124.8 s (ahora dentro del objetivo de 2.5 s) |
| **CLS** | 0 | 0 | Sin cambio |
| **INP** | Sin datos | Sin datos | PageSpeed no lo mide sin datos de usuarios reales |
| **Bytes transferidos** | 31,105 KiB (~30 MB) | 299 kB (~0.3 MB) | ≈ −99 % |

Otros valores: FCP 0.8 s → 0.8 s · TBT 0 ms → 0 ms · Speed Index 3.5 / 2.2 s → 2.1 s.

**Nota sobre el peso final:** la primera medición con DevTools marcó 2.8 MB, pero 2,543 kB eran de `content.css`, cargado por una extensión del navegador y no por el sitio. Repetí la medición en ventana de incógnito (sin extensiones, bajando por toda la página): **9 requests, 299 kB transferred**.

---

## 5. Conclusión breve
* **Cambio con mayor impacto:** la optimización de las imágenes de la galería (`f_auto,q_auto` y ancho acorde, junto con `loading="lazy"`). Esas 5 imágenes sumaban ~31 MB y hoy pesan ~183 kB en WebP. Al bajar tanto el peso, dejaron de competir por el ancho de banda con el hero, y el LCP pasó de 126.9 s a 2.1 s. Como solo medí antes y después de todos los cambios, no puedo separar cuánto aportó el formato/tamaño y cuánto el `lazy`; harían falta mediciones intermedias.
* **Cambio con menor impacto:** `defer` en `script.js` y los atributos `width`/`height`. El script pesa 0.5 kB y TBT ya era 0 ms. En cuanto al CLS, ya era 0 antes de optimizar (las imágenes están debajo del primer viewport), así que no se midió mejora; se dejaron como prevención.
* **Problema que todavía queda pendiente:** en el desglose del LCP final, el mayor tiempo es el retraso en renderizar el elemento (1,140 ms), no la descarga de la imagen. Una posible causa es el CSS que bloquea el render, pero habría que comprobarlo. Además, PageSpeed sigue marcando "Mejora la entrega de imágenes" (131 KiB de ahorro posible) y "Solicitudes de bloqueo de renderización" (330 ms, que ahora viene de `styles.css`). El Desempeño quedó en 99, no en 100.
* **¿Apareció algún problema nuevo?** No en las métricas. Solo noté que el navegador pide `favicon.ico` y devuelve 404 (el sitio no tiene favicon), lo cual no afecta el desempeño.
* **¿Qué harías en una segunda iteración?** Usar `srcset`/`<picture>` para servir imágenes de distinto tamaño según el dispositivo, minificar `styles.css` o poner el CSS crítico en línea, y agregar un favicon.
* **¿Conservarías todos los cambios en producción?** Sí. Ninguno empeoró una métrica. El `loading="lazy"` solo está en imágenes fuera del primer viewport, y el hero (visible al cargar) no lo usa, por lo que no se perjudica el LCP.

---

## 6. Respuestas a las preguntas finales

1. **¿Qué significa LCP y cuál fue el elemento LCP antes y después?**
   LCP (*Largest Contentful Paint*) es el tiempo que tarda en pintarse el elemento de contenido más grande visible en pantalla. Antes: no pude consultar el elemento exacto, porque el reporte inicial ya no estaba disponible cuando redacté esta respuesta. Lo único que quedó registrado es que en ese reporte aparecía marcada la auditoría "Descubrimiento de solicitudes de LCP", que aplica cuando el LCP es una imagen; por eso supongo que fue una imagen (probablemente la del hero), pero no lo pude confirmar. El LCP era de 126.9 s. Después: la imagen del hero (`img.hero__image`, "Campo estelar con una nebulosa azul y violeta") con 2.1 s. En el desglose del LCP final, la descarga del recurso dura solo 20 ms (con 120 ms de retraso previo) y la mayor parte del tiempo es el retraso en la renderización del elemento (1,140 ms).

2. **¿Cómo puede una imagen sin dimensiones explícitas contribuir a CLS?**
   Sin `width` y `height` el navegador no sabe cuánto espacio reservar. Cuando la imagen termina de descargarse, ocupa su tamaño real y empuja hacia abajo el contenido que ya estaba pintado, causando un salto de layout que suma al CLS. En mi medición el CLS fue 0 antes y después porque las imágenes están debajo del primer viewport, pero declarar las dimensiones evita el problema.

3. **¿Por qué `loading="lazy"` es útil en una galería, pero puede ser una mala idea para una imagen principal visible al cargar?**
   En una galería, `lazy` evita descargar las imágenes hasta que el usuario se acerca a ellas, ahorrando datos. En el hero, que se ve apenas carga la página y suele ser el elemento LCP, `lazy` retrasaría su descarga y empeoraría el LCP. Por eso el hero lleva `fetchpriority="high"` y no `lazy`.

4. **Diferencia entre `defer` y `async`. ¿Cuál elegiste y por qué?**
   Ambos descargan el script en paralelo sin bloquear el parser. `async` lo ejecuta apenas termina de descargarse (sin orden garantizado), y puede interrumpir el parseo. `defer` espera a que todo el HTML esté parseado y ejecuta los scripts en orden, justo antes de `DOMContentLoaded`. Elegí `defer` porque `script.js` busca elementos del DOM (menú, botón, contador) que deben existir cuando se ejecute.

5. **¿Qué formato de imagen elegiste y qué comparación de peso obtuviste?**
   Usé la entrega automática de formatos modernos de Cloudinary (`f_auto,q_auto`), que entregó **WebP** (se ve en la columna *Type* de DevTools). Hero: **382 kB** (JPG original) → **111 kB** (WebP), ≈ −71 %. Galería (5 imágenes): ~31 MB → ~183 kB. Peso total de la página: 31,105 KiB → 299 kB (≈ −99 %). El mayor ahorro de bytes estuvo en la galería, no en el hero.

6. **Si el puntaje sube, pero el LCP sigue por encima del objetivo, ¿consideras terminada la optimización?**
   No. El puntaje de desempeño es un promedio ponderado de varias métricas, y un buen resultado en TBT, CLS o FCP puede disimular un LCP malo. El LCP es un Core Web Vital por sí mismo (objetivo: menos de 2.5 s), y es lo que el usuario percibe como "la página ya cargó". Mientras no esté dentro del objetivo, hay que seguir optimizando el elemento LCP. En mi caso el LCP final fue 2.1 s, dentro del objetivo.