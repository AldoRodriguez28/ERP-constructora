# 📦 Modelo de Datos MVP — ERP Construcción (PostgreSQL)

Este documento define el **modelo de datos mínimo viable (MVP)** para el ERP de construcción, alineado estrictamente al flujo definido en `/docs/FLOWS.md`.

❗ **No se inventan procesos fuera del flujo**  
✔ Incluye PK/FK, enums de estado, timestamps y auditoría básica  
✔ Reglas de negocio implementadas a nivel base de datos

---

## 🧭 Flujo soportado (resumen)

1. Alta de obra
2. Alta de proveedor
3. Alta de maquinaria
4. Requisición (ligada a obra)
5. Orden de compra (aprobada desde finanzas)
6. Factura del proveedor
7. Pago
8. Cobro por obra
9. Dashboard financiero por obra

---

## 🔗 Relaciones principales

- **obra** 1—N **requisicion**
- **requisicion** 1—N **requisicion_item**
- **proveedor** 1—N **orden_compra**
- **obra** 1—N **orden_compra**
- **orden_compra** 1—N **orden_compra_item**
- **orden_compra** 1—N **factura**
- **factura** 1—N **pago**
- **obra** 1—N **cobro_obra**
- **rol** 1—N **usuario**
- **maquinaria** N—1 **obra** (asignación opcional)

---

## 📐 Reglas de negocio (obligatorias)

### 1️⃣ Material → Obra obligatorio

- Toda **requisición** y **orden de compra** debe pertenecer a una obra.
- Implementado con `obra_id NOT NULL`.

### 2️⃣ Refacción → Maquinaria obligatoria

- Si un ítem es de tipo `refaccion`, **debe** indicar maquinaria.
- Implementado con `CHECK CONSTRAINT`.

---

## 🧾 Enums de estados

```sql
CREATE TYPE obra_status         AS ENUM ('activa', 'pausada', 'cerrada');
CREATE TYPE maquinaria_status   AS ENUM ('disponible', 'asignada', 'mantenimiento', 'baja');
CREATE TYPE requisicion_status  AS ENUM ('borrador', 'enviada', 'aprobada', 'rechazada', 'cerrada');
CREATE TYPE orden_compra_status AS ENUM ('borrador', 'emitida', 'aprobada', 'cancelada', 'cerrada');
CREATE TYPE factura_status      AS ENUM ('recibida', 'validada', 'rechazada', 'pagada', 'cancelada');
CREATE TYPE pago_status         AS ENUM ('programado', 'enviado', 'confirmado', 'cancelado');
CREATE TYPE cobro_obra_status   AS ENUM ('pendiente', 'facturado', 'cobrado', 'cancelado');
CREATE TYPE item_tipo           AS ENUM ('material', 'refaccion', 'servicio');


👤 Seguridad
Tabla: rol

CREATE TABLE rol (
  id BIGSERIAL PRIMARY KEY,
  nombre TEXT NOT NULL UNIQUE,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now(),
  created_by BIGINT,
  updated_by BIGINT,
  deleted_at TIMESTAMPTZ
);


Tabla: usuario

CREATE TABLE usuario (
  id BIGSERIAL PRIMARY KEY,
  rol_id BIGINT REFERENCES rol(id),
  nombre TEXT NOT NULL,
  email TEXT UNIQUE NOT NULL,
  password_hash TEXT NOT NULL,
  activo BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now(),
  created_by BIGINT REFERENCES usuario(id),
  updated_by BIGINT REFERENCES usuario(id),
  deleted_at TIMESTAMPTZ
);

🏗️ Catálogos base
Tabla: obra

CREATE TABLE obra (
  id BIGSERIAL PRIMARY KEY,
  codigo TEXT UNIQUE NOT NULL,
  nombre TEXT NOT NULL,
  status obra_status DEFAULT 'activa',
  fecha_inicio DATE,
  fecha_fin DATE,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now(),
  created_by BIGINT REFERENCES usuario(id),
  updated_by BIGINT REFERENCES usuario(id),
  deleted_at TIMESTAMPTZ
);


Tabla: proveedor
CREATE TABLE proveedor (
  id BIGSERIAL PRIMARY KEY,
  nombre TEXT NOT NULL,
  rfc TEXT UNIQUE,
  email TEXT,
  telefono TEXT,
  direccion TEXT,
  activo BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now(),
  created_by BIGINT REFERENCES usuario(id),
  updated_by BIGINT REFERENCES usuario(id),
  deleted_at TIMESTAMPTZ
);

Tabla: maquinaria
CREATE TABLE maquinaria (
  id BIGSERIAL PRIMARY KEY,
  codigo TEXT UNIQUE NOT NULL,
  nombre TEXT NOT NULL,
  status maquinaria_status DEFAULT 'disponible',
  obra_id BIGINT REFERENCES obra(id),
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now(),
  created_by BIGINT REFERENCES usuario(id),
  updated_by BIGINT REFERENCES usuario(id),
  deleted_at TIMESTAMPTZ
);

📄 Requisición
Tabla: requisicion
CREATE TABLE requisicion (
  id BIGSERIAL PRIMARY KEY,
  obra_id BIGINT NOT NULL REFERENCES obra(id),
  solicitante_id BIGINT REFERENCES usuario(id),
  status requisicion_status DEFAULT 'borrador',
  notas TEXT,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now(),
  created_by BIGINT REFERENCES usuario(id),
  updated_by BIGINT REFERENCES usuario(id),
  deleted_at TIMESTAMPTZ
);

Tabla: requisicion_item
CREATE TABLE requisicion_item (
  id BIGSERIAL PRIMARY KEY,
  requisicion_id BIGINT REFERENCES requisicion(id) ON DELETE CASCADE,
  tipo item_tipo NOT NULL,
  descripcion TEXT NOT NULL,
  cantidad NUMERIC CHECK (cantidad > 0),
  precio_estimado NUMERIC,
  maquinaria_id BIGINT REFERENCES maquinaria(id),
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now(),
  created_by BIGINT REFERENCES usuario(id),
  updated_by BIGINT REFERENCES usuario(id),
  deleted_at TIMESTAMPTZ,
  CHECK (tipo <> 'refaccion' OR maquinaria_id IS NOT NULL)
);

🧾 Orden de compra
Tabla: orden_compra
CREATE TABLE orden_compra (
  id BIGSERIAL PRIMARY KEY,
  obra_id BIGINT NOT NULL REFERENCES obra(id),
  proveedor_id BIGINT REFERENCES proveedor(id),
  requisicion_id BIGINT REFERENCES requisicion(id),
  status orden_compra_status DEFAULT 'borrador',
  folio TEXT UNIQUE,
  subtotal NUMERIC DEFAULT 0,
  impuestos NUMERIC DEFAULT 0,
  total NUMERIC DEFAULT 0,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now(),
  created_by BIGINT REFERENCES usuario(id),
  updated_by BIGINT REFERENCES usuario(id),
  deleted_at TIMESTAMPTZ
);

Tabla: orden_compra_item
CREATE TABLE orden_compra_item (
  id BIGSERIAL PRIMARY KEY,
  orden_compra_id BIGINT REFERENCES orden_compra(id) ON DELETE CASCADE,
  tipo item_tipo NOT NULL,
  descripcion TEXT,
  cantidad NUMERIC CHECK (cantidad > 0),
  precio_unitario NUMERIC,
  importe NUMERIC,
  maquinaria_id BIGINT REFERENCES maquinaria(id),
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now(),
  created_by BIGINT REFERENCES usuario(id),
  updated_by BIGINT REFERENCES usuario(id),
  deleted_at TIMESTAMPTZ,
  CHECK (tipo <> 'refaccion' OR maquinaria_id IS NOT NULL)
);


🧮 Facturación y pagos
Tabla: factura
CREATE TABLE factura (
  id BIGSERIAL PRIMARY KEY,
  orden_compra_id BIGINT REFERENCES orden_compra(id),
  proveedor_id BIGINT REFERENCES proveedor(id),
  status factura_status DEFAULT 'recibida',
  folio TEXT,
  subtotal NUMERIC,
  impuestos NUMERIC,
  total NUMERIC,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now(),
  created_by BIGINT REFERENCES usuario(id),
  updated_by BIGINT REFERENCES usuario(id),
  deleted_at TIMESTAMPTZ
);

Tabla: pago
CREATE TABLE pago (
  id BIGSERIAL PRIMARY KEY,
  factura_id BIGINT REFERENCES factura(id),
  status pago_status DEFAULT 'programado',
  fecha_pago DATE,
  monto NUMERIC CHECK (monto > 0),
  referencia TEXT,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now(),
  created_by BIGINT REFERENCES usuario(id),
  updated_by BIGINT REFERENCES usuario(id),
  deleted_at TIMESTAMPTZ
);

💰 Cobro por obra
Tabla: cobro_obra
CREATE TABLE cobro_obra (
  id BIGSERIAL PRIMARY KEY,
  obra_id BIGINT REFERENCES obra(id),
  status cobro_obra_status DEFAULT 'pendiente',
  concepto TEXT,
  monto NUMERIC CHECK (monto > 0),
  fecha_cobro DATE,
  referencia TEXT,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now(),
  created_by BIGINT REFERENCES usuario(id),
  updated_by BIGINT REFERENCES usuario(id),
  deleted_at TIMESTAMPTZ
);

```
