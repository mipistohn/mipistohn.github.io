# Roadmap — Mi Pisto HN

Lista de trabajo pendiente y decisiones técnicas por hacer. No es una promesa
de fechas; es la prioridad acordada.

---

## 🔴 Prioridad alta

### Persistencia por registro (de O(n) a O(1) por escritura)

**Problema.** Hoy el estado de cada usuario se guarda como **un solo bloque
cifrado** (un «blob»). Da privacidad total (zero-knowledge), pero cada vez que
se guarda un cambio se **re-cifra y re-sube todo el historial**. El costo de
cada escritura crece con la cantidad de datos (**O(n)**) cuando debería ser
constante (**O(1)**).

**Mediciones** (data con la misma forma que la real; tiempos en una PC. En
Android de gama media/baja el JS corre ~4-10× más lento, y la app cifra dos
veces por guardado: localStorage + IndexedDB):

| Movimientos | Años (a 10/día) | Bloque cifrado | Guardar (PC) | Abrir (PC) | Memoria |
|------------:|----------------:|---------------:|-------------:|-----------:|--------:|
| 1,000       | 0.3             | 0.5 MB         | 12 ms        | 4 ms       | ~8 MB   |
| 3,650       | 1               | 1.7 MB         | 21 ms        | 8 ms       | ~13 MB  |
| 10,000      | 2.7             | 4.5 MB         | 69 ms        | 22 ms      | ~35 MB  |
| 20,000      | 5.5             | 9.1 MB         | 118 ms       | 53 ms      | ~33 MB  |
| 50,000      | 13.7            | 22.7 MB        | 334 ms       | 133 ms     | ~47 MB  |

**Lectura:**
- **Memoria:** no es el problema (<50 MB incluso a 14 años).
- **CPU por guardado:** sí. Con ~10,000 movimientos, anotar un gasto podría
  congelar la pantalla ~0.5-1.4 s en un teléfono barato. Molesto a los ~3-5
  años de uso intenso.
- **Red:** sí. Cada cambio sube el bloque completo y el otro dispositivo lo baja.
- **localStorage** se llena alrededor de ~10,000 movimientos (hay respaldo en
  IndexedDB).

**Lo que NO lo resuelve.** Mudar a una base relacional en un servidor. El
problema es la **granularidad** con que se guarda, no *dónde* viven los datos:
si el cliente sigue bajando y reescribiendo todo, seguiría lento aunque atrás
hubiera Postgres. Además rompería el cifrado de extremo a extremo (el servidor
pasaría a leer los datos).

**La corrección (manteniendo la privacidad):**
1. **Guardar registro por registro** en el dispositivo: almacén en IndexedDB
   con clave `id` e índice por `date`. Anotar un gasto pasa a escribir un solo
   registro cifrado (~400 bytes): **O(1)** en vez de O(n).
2. **No cargar todo el historial en memoria:** solo el periodo actual; los
   saldos se calculan con cortes mensuales precalculados (saldo al cierre
   anterior + movimientos del mes).
3. **Sincronización incremental:** en Supabase, una fila cifrada por registro
   con `updated_at`; sincronizar es pedir «dame lo que cambió desde X» (unos KB).

**Migración sin relanzar ni perder usuarios.** El cambio es del lado del
cliente: al abrir la app, el propio teléfono descifra el bloque viejo, lo parte
en registros y los re-sube. Es un update normal y silencioso, consistente con
el cifrado E2E; no requiere relanzar la app ni perder el avance de la tienda.

**Nota de privacidad.** «Una fila por registro» hace que el servidor vea
*cuántos* registros hay y *cuándo* cambió cada uno (metadatos, nunca el
contenido). Es un intercambio razonable y habitual en apps E2E, pero se decide
a conciencia.

**Estado.** Funciona bien hoy porque una persona tarda años en llegar a esos
volúmenes. Es **deuda técnica real y planificada**: conviene hacerla temprano
(con pocos usuarios), no como reescritura.

**Alternativa considerada.** SQLite compilado a WebAssembly (build oficial de
sqlite.org) sobre OPFS. En contra: en GitHub Pages hay que usar el modo
`opfs-sahpool` (el modo OPFS normal pide cabeceras HTTP que Pages no permite),
y no trae cifrado (habría que cifrar los campos en la app, y entonces SQL no
puede filtrar por esos campos). Por eso se prefiere **IndexedDB por registro**:
más simple y conserva el cifrado.

> Detalle y diagrama en el documento de arquitectura del proyecto.
