# Simulación de la línea de recepción IoT (Python + PostgreSQL)

El documento ".zip" contiene la base de datos `papeleria_db`, así como la simulación del proceso realizada en python.

## Qué se necesita

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

## Archivos

| Archivo | Para qué sirve |
| `main.py` | Arranca: conecta, prepara la base y abre la ventana |
| `config.py` | Conexión de simulación a base de datos|
| `bd.py` | Todas las consultas e inserciones a PostgreSQL |
| `motor.py` | Lo que pasa físicamente en las bandas y cuándo se escribe en la base |
| `planta.py` | El dibujo animado de la planta |
| `paneles.py` | El panel de la orden y las pestañas |
| `interfaz.py` | La ventana principal |
| `estilo.py` | Colores claros y estilos |
| `papeleria_postgresql_v2.sql` | Tu script de la base de datos en sql |
| `preparar_simulacion.sql` |Se ejecuta solo al arrancar |
| `reiniciar_datos.sql` | Opcional, para empezar de cero |
| `requirements.txt` | Dependencia: psycopg2-binary |
