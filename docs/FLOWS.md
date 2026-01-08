# Especificación de Flujo: ERP Construcción (Módulo Operativo)

## 1. Matriz de Pantallas y Datos

| Pantalla             | Campos Mínimos (Input)                                              | Regla de Interfaz                              |
| :------------------- | :------------------------------------------------------------------ | :--------------------------------------------- |
| **Alta Obra**        | Nombre, Centro de Costos, Presupuesto Total, Fecha Inicio.          | Solo editable por Administrador.               |
| **Alta Proveedor**   | RFC, Razón Social, CLABE, Especialidad, Correo de Pagos.            | Validación de RFC obligatorio.                 |
| **Requisición**      | Tipo (Material/Refacción), Item, Cantidad, Unidad, ID Maquinaria\*. | \*ID Maquinaria obligatorio si es "Refacción". |
| **Orden de Compra**  | Proveedor, Precio Unitario, % IVA, Condiciones de Pago.             | Calcula totales automáticamente.               |
| **Registro Factura** | Carga de XML y PDF, UUID (automático), Fecha Vencimiento.           | Match obligatorio con Orden de Compra.         |
| **Programar Pago**   | Fecha de Salida, Prioridad, Cuenta Origen.                          | Bloqueado si la factura no está validada.      |

---

## 2. Estados del Proceso (Workflows)

### A. Requisición

- `NUEVA`: Creada en campo.
- `APROBADA`: Validada por residente de obra.
- `CONVERTIDA`: Ya tiene una Orden de Compra asociada.

### B. Orden de Compra (OC)

- `PENDIENTE`: En espera de validación financiera.
- `AUTORIZADA`: Con firma digital para envío al proveedor.
- `RECIBIDA`: Confirmación de que el material llegó a almacén/obra.

### C. Ciclo de Pago

- `POR REGISTRAR`: Mercancía recibida, falta factura.
- `PROGRAMADO`: Con fecha de pago en calendario de tesorería.
- `PAGADO`: Con comprobante de transferencia ligado.

---

## 3. Reglas de Validación (Lógica de Negocio)

### 1. Validación de Presupuesto (Materiales)

- **Regla:** Al generar una Requisición de tipo "Material", el sistema debe comparar el costo estimado contra el presupuesto disponible de la partida de la Obra.
- **Acción:** Si excede, el sistema dispara una alerta y requiere aprobación de "Gerencia".

### 2. Trazabilidad de Activos (Refacciones)

- **Regla:** Si el tipo es "Refacción", el usuario DEBE seleccionar un equipo del catálogo de maquinaria.
- **Acción:** El costo se carga directamente al historial de mantenimiento de esa máquina, no solo a la obra.

### 3. Validación Fiscal (Facturación)

- **Regla:** El sistema debe extraer el RFC del emisor del XML.
- **Acción:** Si el RFC del XML no coincide con el RFC del Proveedor seleccionado en la OC, el sistema bloquea el registro.

### 4. Integridad de Pagos

- **Regla:** No se puede marcar como "Pagado" si no se ha subido el archivo del Comprobante de Pago (CEP) o captura de transferencia.
- **Acción:** El estado "Pagado" cierra el ciclo y resta el monto del flujo de caja real.

---

## 4. Diagrama de Flujo (Resumen)

[Inicio]
↓
(Alta Obra/Prov)
↓
[Requisición] --(¿Es Refacción?)--> [Asignar a Maquinaria]
↓
[Orden de Compra]
↓
[Aprobación Finanzas] --(Rechazo)--> [Ajustar OC]
↓
[Carga de Factura XML]
↓
[Programación y Pago]
↓
[Fin: Gasto Registrado]
