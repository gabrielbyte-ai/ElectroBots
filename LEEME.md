# Simulación de la línea de recepción IoT (Python + PostgreSQL)

Es la misma simulación de la página web, ahora en Python. Cada lectura de sensor y cada decisión de una persona se guarda en tu base `papeleria_db`.

## Qué necesitas

- Python 3.9 o más nuevo. En Windows, el instalador de python.org ya trae Tkinter, que es la librería de la ventana.
- Tu base `papeleria_db` en PostgreSQL 13 o más nuevo. Sirve la que creaste con `papeleria_postgresql_v1.sql` o una nueva creada con `papeleria_postgresql_v2.sql`, que viene en esta carpeta.

## Cómo correrla

1. Descomprime la carpeta y abre una terminal dentro de ella.
2. Instala la única dependencia:
   - Windows: `py -m pip install -r requirements.txt`
   - macOS o Linux: `python3 -m pip install -r requirements.txt`
3. Arranca la simulación:
   - Windows: `py main.py`
   - macOS o Linux: `python3 main.py`

`config.py` ya trae tus datos de conexión: localhost, puerto 5432, base papeleria_db, usuario postgres y tu contraseña. Cámbialos ahí solo si algo de eso cambia.

## Qué cambia en tu base al arrancar

`main.py` revisa que existan tus tablas y ejecuta `preparar_simulacion.sql`. Puedes correrlo las veces que quieras: no borra nada. Hace esto:

1. Llena `proveedor_productos.codigo_proveedor` de PROV-001 con la numeración `01` a `16` (P-001 → 01, P-002 → 02…), solo donde está vacía. Es el número con el que el ID de cada caja dice qué contiene.
2. Da de alta en `dispositivos_iot` los 11 dispositivos (IR-E1, LB-E3, IR-E3, DESV-D1, IMP-RFID-E5, RFID-E6, IR-E6, DESV-D2, LB-REV, IMP-RFID-REV, RFID-VENTA). Si ya existen, no los toca.
3. Si corriste una versión anterior, corrige dos descripciones: IR-E3 e IR-E6 van «después del lector», e IMP-RFID-E5 pasa a ser una ETIQUETADORA_RFID que imprime, codifica y pega el tag.
4. Agrega VENDIDO a los estados permitidos de `paquetes.estado_validacion`. Solo cambia ese CHECK si todavía no lo acepta.
5. Las cajas que vendiste con la versión anterior quedaron en ALMACENADO aunque tienen su SALIDA. Pasan a VENDIDO y el cambio queda en `auditoria`.

Si creas la base desde cero con `papeleria_postgresql_v2.sql`, ya trae todo esto y este paso no cambia nada. Los costos quedan en NULL.

## papeleria_postgresql_v2.sql

Es tu script v1 con estos cambios, anotados en su encabezado:

- `paquetes.estado_validacion` acepta VENDIDO. Si lo corres sobre una base que ya existe, cambia ese CHECK sin borrar datos.
- Llena la numeración `codigo_proveedor` (01 a 16) de Norma.
- Da de alta los 11 dispositivos IoT de la línea en `dispositivos_iot`.

## Cómo se usa

- **Panel izquierdo:**
  - la orden de compra: numeración de producto y cajas;
  - las fallas a simular: código de barras dañado (cajas) y tag RFID pegado en la caja equivocada (veces);
  - lo que realmente entrega el proveedor: cajas que faltan y cajas excedentes.

  Pulsa **Iniciar recepción**.
- **Planta:**
  - el camión descarga en la banda de recepción, donde el IR de E1 cuenta;
  - la persona de E2 carga la línea, que pasa por el lector de barras y su IR (E3), el desviador D1 (E4), la impresora-etiquetadora RFID automática (E5), y la antena RFID, su IR y el desviador D2 (E6), hasta llegar al almacén;
  - en E3 y E6 el IR va **después** del lector: el lector lee mientras la caja pasa y el IR que sigue confirma la caja. Si el IR detecta una caja y el lector no entregó nada, es una caja sin lectura;
  - E5 es una sola máquina, sin operador: imprime y codifica el tag con el EPC del paquete y lo pega en la caja. La falla «tag RFID pegado en la caja equivocada» es un desfase de esa máquina: dos cajas quedan cada una con el tag de la otra. La máquina no se entera; lo descubre la antena de E6 y las dos cajas van a revisión B.

  Haz clic en una caja para ver qué sabe la base de ella. **Ver verdad física** muestra los tags cruzados, que ningún sensor ve.
- **Pestañas:**
  - **Tareas:** aprobar o rechazar la entrega, resolver las revisiones A y B, y cerrar la recepción.
  - **Conciliación:** compara la orden con lo que llegó al almacén, junto al inventario por ubicación.
  - **Base de datos:** muestra en vivo recepciones, detalle_recepcion, existencias, eventos_iot, incidencias, ordenes_trabajo, dispositivos_iot y productos.
  - **Venta de producto:** escanear un tag y vender la caja.
  - **Bitácora:** el registro de todo lo que pasó.
- **Velocidad:** el deslizador va de lento (izquierda) a rápido (derecha).
- **Atajos:** Espacio ejecuta o pausa, y la flecha derecha avanza un paso.

## La tabla «existencias» de la pestaña Base de datos

Es tu tabla `paquetes`: una fila por cada caja registrada, con su tag y su estado. En PostgreSQL se sigue llamando `paquetes`, porque tu base ya tiene otra tabla llamada `existencias` (el saldo por producto y ubicación) y dos tablas no pueden llamarse igual. Solo cambia el nombre que ves en la simulación.

| Columna | De dónde sale | Qué significa |
|---|---|---|
| `epc_rfid` | paquetes | El EPC grabado en el tag RFID de la caja |
| `id_producto`, `producto` | detalle_paquete y productos | Qué contiene la caja |
| `estado_validacion` | paquetes | PENDIENTE al registrarse en E3, VALIDADO en E6, ALMACENADO en el almacén y VENDIDO al venderla. RETENIDO mientras está en revisión B y DEVUELTO si regresó al proveedor |
| `id_recepcion` | paquetes | La recepción en la que entró la caja (abajo se explica) |
| `folio_recepcion` | recepciones | El folio de esa recepción, para leerlo fácil |
| `fecha_registro` | paquetes | Cuándo se registró la caja |
| `id_paquete` | paquetes | El identificador interno de la caja |

Arriba salen las cajas que se movieron al último, así que la caja que acabas de vender aparece primero, marcada en verde.

### Qué es `id_recepcion`

Es la llave que une cada caja con la recepción, es decir, con el camión en el que llegó.

1. Cuando llega un camión, la simulación crea una fila en `recepciones`. PostgreSQL le pone un identificador único con `gen_random_uuid()`, por ejemplo `112790f0-5cb0-4383-b652-e34ccf74d693`.
2. Cada caja de esa descarga que se registra como paquete guarda ese mismo valor en `paquetes.id_recepcion`. Todas las cajas de un mismo camión tienen el mismo `id_recepcion`.
3. Es una llave foránea (`REFERENCES recepciones(id_recepcion)`): la base no deja guardar una caja con una recepción que no existe.

Con esa columna sabes de qué proveedor, documento y fecha viene cada caja, y la conciliación cuenta cuántas cajas llegaron en cada recepción. El UUID es para la base; para las personas está el folio (`REC-20261125-001`), que la simulación muestra al lado en `folio_recepcion`. Por ejemplo:

```sql
SELECT r.folio, p.estado_validacion, COUNT(*) AS cajas
FROM paquetes p
JOIN recepciones r ON r.id_recepcion = p.id_recepcion
GROUP BY r.folio, p.estado_validacion
ORDER BY r.folio;
```

## Qué se guarda en cada tabla

| Momento | Lo que se escribe |
|---|---|
| Llega el camión (E1) | `recepciones` en BORRADOR y `detalle_recepcion` con `cantidad_esperada` = cajas y `costo_unitario` NULL |
| Cada caja que cuenta el IR de E1 | `eventos_iot` DETECCION_IR (IR-E1) |
| El conteo no cuadra con la orden | `incidencias` DIFERENCIA_CONTEO_RECEPCION |
| Aprobar o rechazar la entrega | `revisiones_humanas` y `recepciones` en EN_PROCESO o CANCELADA |
| El lector de barras (E3) lee el ID | `eventos_iot` LECTURA_CODIGO en PENDIENTE |
| El IR que sigue al lector confirma la caja | `eventos_iot` DETECCION_IR (IR-E3). Si el ID es válido, la lectura pasa a PROCESADO y se crean `paquetes` con su `epc_rfid`, `detalle_paquete`, el movimiento RECEPCION a REC-01 y `existencias` |
| El IR detecta una caja sin lectura, o el ID tiene un problema | La lectura, si la hubo, pasa a RECHAZADO y se abre la incidencia (SIN_LECTURA_CODIGO, ID_DUPLICADO, CAJA_EXCEDENTE, PRODUCTO_NO_ESPERADO…) |
| Desviador D1 | `eventos_iot` DESVIO_A_REVISION. La incidencia pasa a EN_REVISION |
| Revisión A | `revisiones_humanas` (AJUSTAR, APROBAR o DEVOLVER). El paquete se crea solo si la caja se acepta |
| Impresora-etiquetadora RFID (E5) | `eventos_iot` ETIQUETADO_RFID (IMP-RFID-E5) con el EPC del tag que imprime, codifica y pega |
| La antena RFID (E6) lee el tag | `eventos_iot` LECTURA_RFID en PENDIENTE |
| El IR que sigue a la antena confirma la caja | `eventos_iot` DETECCION_IR (IR-E6). La lectura se compara con el EPC esperado por posición: pasa a PROCESADO y el paquete queda VALIDADO, o pasa a RECHAZADO y se abre la incidencia EPC_NO_COINCIDE, EPC_SIN_LECTURA o EPC_DUPLICADO |
| Desviador D2 | Paquete RETENIDO y movimiento CUARENTENA de REC-01 a CUA-01 |
| Revisión B, reetiquetar | `eventos_iot` REIMPRESION_RFID, movimiento LIBERACION_CUARENTENA de CUA-01 a ALM-01 y paquete ALMACENADO |
| Revisión B, devolver | Paquete DEVUELTO y movimiento AJUSTE_SALIDA desde CUA-01 |
| Llega al almacén | Movimiento ALMACENAMIENTO de REC-01 a ALM-01 y paquete ALMACENADO |
| Termina la línea | `incidencias` FALTANTE_PRODUCTO o EXCEDENTE_RECIBIDO, y `recepciones` VALIDADA o CON_INCIDENCIAS |
| Cerrar la recepción | `recepciones` CERRADA. Los faltantes y excedentes siguen PENDIENTE en `incidencias`, para compras |
| Venta | `eventos_iot` LECTURA_RFID_VENTA, `ordenes_trabajo` SURTIDO COMPLETADA con su detalle, movimiento SALIDA desde ALM-01 y el paquete pasa a VENDIDO |

Cada cambio de estado deja también un registro en `auditoria`. Cada operación es una sola transacción: si la base rechaza algo, no queda nada a medias.

## Decisiones que tomé con tu esquema

1. **1 caja = 1 unidad.** `cantidad_esperada`, `cantidad_declarada` y las cantidades de los movimientos cuentan cajas.
2. **El ID de cada caja** es la numeración de 2 dígitos más 6 dígitos de serie, por ejemplo `03001013` para una caja de P-003. Se guarda en `eventos_iot.datos_originales` y con eso se detectan IDs repetidos.
3. **El EPC del tag** es `id_producto-AAMMDD-consecutivo`, por ejemplo `P-003-261125-00008`, y va en `paquetes.epc_rfid`.
4. **Una caja desviada en revisión A** no entra al inventario hasta que alguien la acepta. Si se devuelve, nunca fue paquete.
5. **Devolver un paquete ya registrado** usa AJUSTE_SALIDA desde CUA-01, porque en tu esquema DEVOLUCION suma existencias.
6. **La venta** es una orden de trabajo SURTIDO con su SALIDA, y el paquete pasa a VENDIDO.
7. **«existencias» en la pestaña Base de datos** es la tabla `paquetes`. En la base no se renombra.
8. **No se usan** `sincronizacion_outbox` ni los costos.

## Empezar de cero

`reiniciar_datos.sql` es opcional y la simulación nunca lo ejecuta. Borra **todas** las recepciones, paquetes, eventos, movimientos, incidencias, revisiones, órdenes de trabajo y auditoría de la base, y pone las existencias en 0. No toca proveedores, productos, ubicaciones ni dispositivos. Córrelo desde pgAdmin o con:

```
psql -U postgres -d papeleria_db -f reiniciar_datos.sql
```

## Si algo falla

- **"No se pudo conectar":** revisa los datos de `config.py` y que el servicio de PostgreSQL esté encendido. En Windows, un error `UnicodeDecodeError` al conectar casi siempre es la contraseña o el usuario.
- **"Faltan tablas":** corre primero `papeleria_postgresql_v2.sql` en la base que pusiste en `config.py`.
- **`No module named psycopg2`:** instala las dependencias (paso 2).
- **`No module named tkinter` en Linux:** `sudo apt install python3-tk`.

## Archivos

| Archivo | Para qué sirve |
|---|---|
| `main.py` | Arranca: conecta, prepara la base y abre la ventana |
| `config.py` | Conexión (ya con tus datos), operador, ubicaciones y orden de ejemplo |
| `bd.py` | Todas las consultas e inserciones a PostgreSQL |
| `motor.py` | Lo que pasa físicamente en las bandas y cuándo se escribe en la base |
| `planta.py` | El dibujo animado de la planta |
| `paneles.py` | El panel de la orden y las pestañas |
| `interfaz.py` | La ventana principal |
| `estilo.py` | Colores claros y estilos |
| `papeleria_postgresql_v2.sql` | Tu script de la base con el estado VENDIDO, la numeración y los dispositivos |
| `preparar_simulacion.sql` | Numeración, dispositivos y estado VENDIDO. Se ejecuta solo al arrancar |
| `reiniciar_datos.sql` | Opcional, para empezar de cero |
| `requirements.txt` | Dependencia: psycopg2-binary |
