# Software FJ – Sistema de Gestión de Clientes, Servicios y Reservas

> Proyecto académico de Programación Orientada a Objetos y Manejo Avanzado de Excepciones.
> **Un solo archivo standalone (`software_fj.python`), sin bases de datos.**

---

## 1. Descripción general

Software FJ es una empresa que ofrece tres tipos de servicios:

| Servicio | Unidad de cobro | Regla particular |
|---|---|---|
| Reserva de salas | hora (1–12) | Recargo de $15/h si incluye proyector |
| Alquiler de equipos | día (1–30) | Control de stock, depósito y 10 % de descuento desde 7 días |
| Asesorías especializadas | hora (1–8) | Multiplicador por nivel: junior ×1.0, senior ×1.5, experto ×2.0 |

El sistema administra **clientes**, **servicios** y **reservas** completamente en memoria
(objetos y listas). El único archivo que se escribe es **`logs.txt`**, donde se registran
todos los eventos y errores.

---

## 2. Requisitos y ejecución

- Python **3.8 o superior**.
- Sin dependencias externas (solo biblioteca estándar).

```bash
python software_fj.python
```

> El intérprete de Python ejecuta el archivo sin importar su extensión, por lo que
> `software_fj.python` funciona igual que cualquier script. Al finalizar se muestra
> un reporte en consola y se genera/actualiza `logs.txt` en el mismo directorio.

---

## 3. Arquitectura del programa

```
EntidadBase (ABC)
├── Cliente
├── Servicio (ABC)
│   ├── ReservaSala
│   ├── AlquilerEquipo
│   └── AsesoriaEspecializada
└── Reserva  ──► EstadoReserva (Enum: PENDIENTE, CONFIRMADA, PROCESADA, CANCELADA)

SistemaGestion (fachada) ──► listas internas de clientes, servicios y reservas
SoftwareFJError (jerarquía de excepciones personalizadas)
```

### 3.1 Secciones del archivo `software_fj.python`

| Sección | Contenido |
|---|---|
| 1 | Configuración del logger (`logs.txt`) y función `registrar_excepcion()` |
| 2 | Jerarquía de excepciones personalizadas con códigos `FJ-xxx` |
| 3 | `EntidadBase`: clase abstracta general (id único, fecha, `validar()`, `describir()`) |
| 4 | `Cliente`: encapsulación con atributos privados y validación por *properties* |
| 5 | `Servicio`: clase abstracta + métodos sobrecargados de costo |
| 6 | Servicios especializados (herencia y polimorfismo) |
| 7 | `Reserva`: ciclo de vida con confirmación, procesamiento y cancelación |
| 8 | `SistemaGestion`: orquestación tolerante a fallos con *rollback* de stock |
| 9 | `simular()`: 20 operaciones válidas e inválidas |
| 10 | Punto de entrada con red de seguridad `try/except/finally` |

---

## 4. Principios POO aplicados

### Abstracción
- `EntidadBase` define el contrato común (`validar`, `describir`).
- `Servicio` define el contrato polimórfico (`calcular_costo`, `validar_parametros`,
  `describir`, `unidad_tiempo`). No puede instanciarse directamente.

### Herencia
- `Cliente`, `Servicio` y `Reserva` heredan de `EntidadBase`.
- `ReservaSala`, `AlquilerEquipo` y `AsesoriaEspecializada` heredan de `Servicio`.

### Polimorfismo
- Cada servicio sobrescribe `calcular_costo()`, `describir()` y `validar_parametros()`
  con sus propias reglas. `SistemaGestion` y `Reserva` trabajan con `Servicio`
  sin conocer el tipo concreto.
- `registrar_servicio(clase, *args)` crea cualquier subclase de forma genérica.

### Encapsulación
- `Cliente` usa atributos privados (`__nombre`, `__correo`, `__telefono`, `__documento`,
  `__historial`) y *properties* con validación por expresiones regulares.
- `historial` devuelve una **copia defensiva** de la lista.
- `Servicio` y `Reserva` exponen atributos protegidos solo mediante *properties*.

### Sobrecarga de métodos
1. **Por parámetros opcionales:**
   `calcular_costo_con_impuesto(duracion, iva=0.15, descuento=0.0)`
2. **Por tipo de argumento** (`functools.singledispatchmethod`):
   `calcular_costo_total(parametro, duracion)` acepta:
   - `None` → costo neto + IVA por defecto
   - `int` / `float` → porcentaje de descuento (0–100)
   - `dict` → opciones `{"iva": 0.12, "descuento": 0.1, "cupon": "FJ10"}`
   - cualquier otro tipo → `CalculoError`

---

## 5. Ciclo de vida de una reserva

```
crear_reserva()
    │  busca cliente → busca servicio → construye Reserva (valida disponibilidad
    │  y duración) → descuenta stock (si es equipo) → confirmar()
    │  Si algo falla después de descontar stock → liberar_unidades() (rollback)
    ▼
PENDIENTE ──confirmar()──► CONFIRMADA ──procesar()──► PROCESADA
    │                          │
    └────────cancelar()────────┘──► CANCELADA (libera stock si es equipo)
```

Reglas:
- Solo se **confirma** una reserva `PENDIENTE`.
- Solo se **procesa** una reserva `CONFIRMADA` con costo > 0.
- No se puede **cancelar** una reserva `PROCESADA` ni una ya `CANCELADA`.

---

## 6. Manejo de excepciones

### 6.1 Jerarquía personalizada

| Excepción | Código | Uso |
|---|---|---|
| `SoftwareFJError` | FJ-000 | Base de todo el sistema |
| `ValidacionError` | FJ-100 | Datos inválidos o parámetros faltantes |
| `ClienteError` | FJ-200 | Base de errores de cliente |
| `ClienteDuplicadoError` | FJ-201 | Documento ya registrado |
| `ClienteNoEncontradoError` | FJ-202 | Cliente inexistente |
| `ServicioError` | FJ-300 | Base de errores de servicio |
| `ServicioNoDisponibleError` | FJ-301 | Deshabilitado o sin stock |
| `ServicioNoEncontradoError` | FJ-302 | Servicio inexistente |
| `ReservaError` | FJ-400 | Base de errores de reserva |
| `ReservaInvalidaError` | FJ-401 | Parámetros de reserva incorrectos |
| `OperacionNoPermitidaError` | FJ-402 | Transición de estado inválida |
| `CalculoError` | FJ-500 | IVA/descuento fuera de rango, total negativo, tipo no soportado |

Los códigos `FJ-xxx` son una convención propia del proyecto (prefijo = nombre de la
empresa, centena = categoría, igual que los códigos HTTP) para filtrar errores en el log.

### 6.2 Estructuras utilizadas

| Estructura | Dónde |
|---|---|
| `try/except` | `procesar_reserva`, `cancelar_reserva`, `cotizar` |
| `try/except/else` | `registrar_cliente`, `registrar_servicio`, `Reserva.confirmar` |
| `try/except/finally` | `crear_reserva`, `Reserva.procesar`, bloque `__main__` |
| Encadenamiento (`raise ... from`) | `Reserva.validar`, `Reserva.confirmar`, `registrar_servicio` (TypeError → ValidacionError) |

**Ningún error detiene la ejecución**: la fachada captura, registra en `logs.txt`,
incrementa el contador de operaciones fallidas y continúa con la siguiente operación.

---

## 7. Cómo se probaron los errores que aparecen en `logs.txt`

Las pruebas se realizaron **desde la propia función `simular()`** de
`software_fj.python`: cada error del log fue provocado intencionalmente con una
entrada inválida controlada, y se verificó que (a) el sistema no se detuviera,
(b) el error quedara registrado con su código `FJ-xxx` y (c) el resto de operaciones
continuara con normalidad.

| Paso | Entrada de prueba usada en `simular()` | Qué se quería comprobar | Línea resultante en `logs.txt` |
|---|---|---|---|
| 3 | `registrar_cliente("Carla Ruiz", "carla@@mail", ...)` | Regex de correo del setter `correo` | `ValidacionError [FJ-100] Correo electrónico inválido` |
| 4 | `registrar_cliente("Pedro 123", ...)` | Setter `nombre` rechaza dígitos | `ValidacionError [FJ-100] El nombre no puede contener dígitos` |
| 5 | Segundo cliente con documento `1712345678` | Detección de duplicados antes de construir el objeto | `ClienteDuplicadoError [FJ-201]` |
| 7a | `registrar_servicio(ReservaSala, "Sala Negativa", -10.0, 5)` | Validación de precio en el constructor de `Servicio` | `ValidacionError [FJ-100] El precio base debe ser positivo (-10.0)` |
| 7b | `registrar_servicio(AsesoriaEspecializada, ..., "master")` | Validación de nivel en la subclase | `ValidacionError [FJ-100] Nivel 'master' inválido` |
| 7c | `registrar_servicio(AlquilerEquipo, "Proyector")` (faltan 3 argumentos) | Captura de `TypeError` de Python y encadenamiento al dominio | `ValidacionError [FJ-100] Parámetros faltantes \| CAUSA: TypeError: missing 3 required positional arguments` |
| 10 | `crear_reserva("9999999999", "Sala Pamplona", 2)` | `buscar_cliente` con documento inexistente | `ClienteNoEncontradoError [FJ-202]` |
| 11 | `crear_reserva(..., "Laptop Dell", 3, cantidad_equipos=1)` después de que el paso 9 agotara las 2 unidades | Servicio marcado `disponible=False` por stock 0 | `ServicioNoDisponibleError [FJ-301] El servicio 'Laptop Dell' no está disponible` |
| 12 | `crear_reserva(..., "Asesoría Legacy", 2)` (servicio creado con `disponible=False`) | Verificación de disponibilidad en el constructor de `Reserva` | `ServicioNoDisponibleError [FJ-301] no está disponible` |
| 13 | `crear_reserva(..., "Sala Pamplona", 20)` | Duración fuera del rango 1–12 h con **encadenamiento** `raise ... from` | `ReservaInvalidaError [FJ-401] \| CAUSA: ValidacionError [FJ-100] entre 1 y 12 horas (solicitado: 20)` |
| 14 | `crear_reserva(..., "Asesoría Cloud", 2, descuento_pct=150)` | Cálculo inconsistente encadenado desde `confirmar()` | `ReservaInvalidaError [FJ-401] No fue posible calcular el costo \| CAUSA: CalculoError [FJ-500] Descuento inconsistente: 150 %` |
| 16a | `procesar_reserva(r1)` por segunda vez | Transición de estado no permitida | `OperacionNoPermitidaError [FJ-402] Solo se procesan reservas CONFIRMADAS (actual: PROCESADA)` |
| 16b | `procesar_reserva(None)` | Manejo de reserva inexistente | `ReservaError [FJ-400] No se puede procesar una reserva inexistente (None)` |
| 17 | `cancelar_reserva(r1, ...)` con `r1` ya procesada | Regla de negocio: no cancelar lo procesado | `OperacionNoPermitidaError [FJ-402] ya fue PROCESADA y no puede cancelarse` |
| 19a | `cotizar("Sala Pamplona", 2, "veinte")` | Tipo no registrado en `singledispatchmethod` | `CalculoError [FJ-500] Tipo de parámetro no soportado: str` |
| 19b | `cotizar("Asesoría Cloud", 3, 120.0)` | Rama `float` de la sobrecarga con descuento > 100 | `CalculoError [FJ-500] Descuento inconsistente: 120.0 %` |
| 20 | `sala.calcular_costo_con_impuesto(2, iva=1.5)` | Parámetro opcional `iva` fuera de rango, capturado en `simular()` | `CalculoError [FJ-500] IVA fuera de rango: 1.5` |

### 7.1 Eventos no-error relevantes del log

| Nivel | Línea | Qué la produce |
|---|---|---|
| `WARNING` | `Reserva ... CANCELADA \| motivo: El cliente cambió de fecha` | Paso 17: `cancelar_reserva(r2, ...)`; devuelve 2 laptops al stock |
| `DEBUG` | `Intento de reserva finalizado: ...` | Bloque `finally` de `crear_reserva`; aparece tanto en éxito como en fallo |
| `DEBUG` | `Fin de intento de procesamiento ... (estado final: PROCESADA)` **antes** de un `ERROR [FJ-402]` | Bloque `finally` de `Reserva.procesar()`: se ejecuta aunque la excepción se propague a la fachada |
| `INFO` | `Reporte final \| clientes=2 servicios=4 reservas=4 exitosas=17 fallidas=16` | `SistemaGestion.reporte()` |

### 7.2 Verificación manual de los montos del log

| Reserva / cotización | Fórmula | Total |
|---|---|---|
| Sala Pamplona, 3 h, proyector, 10 % dto. | $$(25\cdot3 + 15\cdot3)\cdot0.90\cdot1.15$$ | $124.20 |
| Laptop Dell, 7 días, 2 unidades, depósito $50 | $$(12.5\cdot7\cdot2 + 50)\cdot0.90\cdot1.15$$ | $232.87 (*) |
| Asesoría Cloud, 4 h, experto | $$40\cdot4\cdot2.0\cdot1.15$$ | $368.00 |
| Laptop Dell, 2 días, 1 unidad | $$(12.5\cdot2 + 50)\cdot1.15$$ | $86.25 |
| Cotización None | $$80\cdot1.15$$ | $92.00 |
| Cotización 25 % | $$80\cdot0.75\cdot1.15$$ | $69.00 |
| Cotización dict (IVA 12 %, cupón FJ10) | $$80\cdot0.90\cdot1.12$$ | $80.64 |

(*) El valor exacto es 232.875; `round()` devuelve 232.87 por la representación binaria
de `float`. Es comportamiento estándar de Python, no un error del sistema.

### 7.3 Comandos útiles para revisar el log

```bash
grep -c ERROR logs.txt        # cantidad de errores controlados
grep "CAUSA:" logs.txt        # errores con encadenamiento de excepciones
grep "FJ-401" logs.txt        # filtrar por código de error
```

En Windows (PowerShell): `Select-String ERROR logs.txt`.

---

## 8. Historial de correcciones

| Versión | Cambio |
|---|---|
| 1.0 | Versión inicial del sistema standalone |
| 1.1 | **Fix en `crear_reserva`:** la `Reserva` se construía después de descontar stock, por lo que al agotar las unidades el servicio quedaba `disponible=False` y una reserva válida fallaba. Se reordenó el flujo (validar → descontar → confirmar) y se agregó *rollback* con `liberar_unidades()` ante cualquier excepción. Resultado en el log: `reservas=2/exitosas=13/fallidas=20` → `reservas=4/exitosas=17/fallidas=16`. |

---

## 9. Cómo extender el sistema

1. **Nuevo servicio:** heredar de `Servicio` e implementar `calcular_costo`,
   `validar_parametros`, `describir` y `unidad_tiempo`; registrarlo con
   `sistema.registrar_servicio(NuevaClase, ...)`.
2. **Nueva excepción:** heredar de la rama adecuada de `SoftwareFJError`
   y asignar un código `FJ-xxx` siguiendo la numeración por categoría.
3. **Nueva variante de costo:** registrar otro tipo con
   `@Servicio.calcular_costo_total.register(tipo)`.

---
