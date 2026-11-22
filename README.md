# Software FJ – Sistema de Gestión de Clientes, Servicios y Reservas

> Proyecto académico de Programación Orientada a Objetos y Manejo Avanzado de Excepciones.
> **Un solo archivo standalone (`software_fj.py`), sin bases de datos.**

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
python software_fj.py
```

Al finalizar se muestra en consola un reporte y se genera/actualiza `logs.txt`
en el mismo directorio.

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

### 3.1 Secciones del archivo `software_fj.py`

| Sección | Contenido |
|---|---|
| 1 | Configuración del logger (`logs.txt`) y función `registrar_excepcion()` |
| 2 | Jerarquía de excepciones personalizadas con códigos `FJ-xxx` |
| 3 | `EntidadBase`: clase abstracta general (id único, fecha, `validar()`, `describir()`) |
| 4 | `Cliente`: encapsulación con atributos privados y validación por *properties* |
| 5 | `Servicio`: clase abstracta + métodos sobrecargados de costo |
| 6 | Servicios especializados (herencia y polimorfismo) |
| 7 | `Reserva`: ciclo de vida con confirmación, procesamiento y cancelación |
| 8 | `SistemaGestion`: orquestación tolerante a fallos |
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
    │  busca cliente → busca servicio → reserva stock (si es equipo)
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

### 6.2 Estructuras utilizadas

| Estructura | Dónde |
|---|---|
| `try/except` | `procesar_reserva`, `cancelar_reserva`, `cotizar` |
| `try/except/else` | `registrar_cliente`, `registrar_servicio`, `Reserva.confirmar` |
| `try/except/finally` | `crear_reserva`, `Reserva.procesar`, bloque `__main__` |
| Encadenamiento (`raise ... from`) | `Reserva.validar`, `Reserva.confirmar`, `registrar_servicio` (TypeError → ValidacionError) |

### 6.3 Formas de generar un error

| Categoría | Ejemplos que lo disparan |
|---|---|
| **Datos inválidos de cliente** | Nombre vacío/con dígitos, correo mal formado, teléfono o documento fuera de rango |
| **Cliente duplicado / inexistente** | Mismo documento dos veces; reservar con documento no registrado |
| **Servicio mal creado** | Precio ≤ 0, capacidad 0, stock negativo, nivel desconocido, clase que no es `Servicio` |
| **Parámetros faltantes** | `registrar_servicio(AlquilerEquipo, "Proyector")` sin el resto de argumentos |
| **Servicio no disponible** | `disponible=False` o stock agotado |
| **Reserva incorrecta** | Duración fuera de rango, duración no entera, cliente/servicio no válidos |
| **Cálculos inconsistentes** | Descuento > 100 %, IVA > 1, tipo de parámetro no soportado |
| **Operación no permitida** | Procesar dos veces, cancelar una procesada, confirmar una no pendiente |
| **Reserva inexistente** | Procesar o cancelar `None` (resultado de una reserva fallida) |
| **Error inesperado** | Cualquier excepción no prevista es capturada por `except Exception` en la fachada y en `__main__` |

**Ningún error detiene la ejecución**: la fachada captura, registra en `logs.txt`,
incrementa el contador de operaciones fallidas y continúa con la siguiente operación.

---

## 7. Archivo `logs.txt`

Formato de cada línea:

```
AAAA-MM-DD HH:MM:SS | NIVEL    | mensaje
```

| Nivel | Qué registra |
|---|---|
| `INFO` | Inicio/fin del sistema, registros exitosos, confirmaciones, procesamientos, reporte |
| `WARNING` | Cancelaciones de reservas |
| `DEBUG` | Auditoría de bloques `finally` (fin de intentos) |
| `ERROR` | Toda excepción controlada, con su `CAUSA:` encadenada si existe |

Ejemplo:

```
2026-11-22 13:15:02 | ERROR    | Reserva de 'Sala Quito' para doc 0923456789 -> ReservaInvalidaError: [FJ-401] Parámetros de reserva incorrectos para 'Sala Quito'. | CAUSA: ValidacionError: [FJ-100] La sala se reserva entre 1 y 12 horas (solicitado: 20).
```

El archivo se abre en modo *append*, por lo que conserva el historial de ejecuciones.

---

## 8. Simulación incluida (20 pasos)

| Pasos | Qué demuestra |
|---|---|
| 1–2 | Registro válido de clientes |
| 3–5 | Clientes inválidos: correo, nombre con dígitos, duplicado |
| 6 | Creación correcta de los tres tipos de servicio (+ uno deshabilitado) |
| 7 | Servicios incorrectos: precio negativo, nivel inválido, parámetros faltantes |
| 8–9 | Reservas exitosas de sala y equipo (agota stock) |
| 10–14 | Reservas fallidas: cliente inexistente, sin stock, deshabilitado, duración fuera de rango, descuento inconsistente |
| 15 | Reserva exitosa de asesoría |
| 16 | Procesamiento válido, repetido (no permitido) y de `None` |
| 17 | Cancelación válida y cancelación de una reserva procesada (no permitida) |
| 18 | Nueva reserva tras liberar stock por cancelación |
| 19–20 | Métodos sobrecargados con tipos válidos e inválidos |

Al final se imprime un **reporte** con clientes, servicios, reservas, ingresos
procesados y el conteo de operaciones exitosas/fallidas.

---

## 9. Cómo extender el sistema

1. **Nuevo servicio:** crear una clase que herede de `Servicio` e implemente
   `calcular_costo`, `validar_parametros`, `describir` y `unidad_tiempo`.
   Registrarlo con `sistema.registrar_servicio(NuevaClase, ...)`.
2. **Nueva excepción:** heredar de la rama adecuada de `SoftwareFJError`
   y asignar un código `FJ-xxx`.
3. **Nueva variante de costo:** registrar otro tipo con
   `@Servicio.calcular_costo_total.register(tipo)`.

---

## 10. Estructura sugerida del repositorio

```
software-fj/
├── software_fj.py   # aplicación completa (standalone)
├── README.md          # este documento
└── logs.txt         # generado automáticamente al ejecutar
```
