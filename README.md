# InventPro — Sistema Profesional de Gestión de Inventarios (SaaS)

Sistema multi-empresa (multi-tenant) listo para vender por suscripción a **USD 30/mes**.
Cada empresa que se registra tiene sus propios datos completamente aislados: usuarios, productos,
bodegas, movimientos, compras y ventas.

## 1. ¿Qué incluye?

- **Autenticación y roles**: OWNER, ADMIN, MANAGER, EMPLOYEE, con permisos diferenciados por acción.
- **Catálogo de productos**: SKU, código de barras, categorías, unidad de medida, precios de costo/venta, IVA, stock mínimo/máximo, imagen.
- **Multi-bodega**: stock independiente por bodega/almacén, con traslados entre bodegas.
- **Kardex / trazabilidad completa**: cada entrada, salida, ajuste, devolución y traslado queda registrado con usuario, fecha y motivo.
- **Órdenes de compra**: creación, recepción parcial o total (actualiza el stock automáticamente), cancelación.
- **Órdenes de venta**: valida disponibilidad antes de confirmar, reserva stock, despacho parcial/total, cancelación con liberación de reserva.
- **Proveedores y clientes**: gestión básica con NIT, contacto, dirección.
- **Panel de control (dashboard)**: valor total del inventario, alertas de stock bajo, órdenes abiertas, movimientos recientes.
- **Alertas de stock bajo**: automáticas según el mínimo configurado por producto.
- **Auditoría**: registro de quién crea/edita/elimina en el sistema.
- **Suscripción con Stripe**: checkout, portal de autogestión, webhooks que activan/suspenden el acceso según el estado del pago. Periodo de prueba de 14 días incluido.
- **Seguridad**: contraseñas con bcrypt, JWT, rate limiting, Helmet, aislamiento estricto de datos por empresa en cada consulta.

## 2. Arquitectura técnica

```
inventpro/
├── backend/     API REST — Node.js + Express + TypeScript + Prisma ORM + PostgreSQL
└── frontend/    React + Vite + TailwindCSS (SPA)
```

- **Base de datos**: PostgreSQL (relacional, ideal para este dominio con transacciones críticas de stock).
- **ORM**: Prisma, con el esquema completo en `backend/prisma/schema.prisma`.
- **Multi-tenancy**: cada tabla de negocio tiene `companyId`; todas las consultas del backend filtran por la empresa del usuario autenticado (nunca se confía en el `companyId` que venga del cliente).
- **Transacciones**: toda operación que mueve stock (compras, ventas, ajustes, traslados) se ejecuta dentro de una transacción de base de datos para evitar inconsistencias.
- **Pagos**: Stripe Billing (modo suscripción), con webhook para mantener sincronizado el estado de la cuenta.

## 3. Cómo correrlo en tu máquina (desarrollo)

### Requisitos
- Node.js 20+
- Docker (para levantar Postgres fácilmente) o una base PostgreSQL propia
- Una cuenta de Stripe (modo test para desarrollar)

### Pasos

```bash
# 1. Base de datos
docker compose up -d

# 2. Backend
cd backend
cp .env.example .env       # edita DATABASE_URL, JWT_SECRET, claves de Stripe
npm install
npm run prisma:migrate     # crea las tablas
npm run seed                # (opcional) datos de ejemplo: admin@demo.com / Demo1234!
npm run dev                 # http://localhost:4000

# 3. Frontend (en otra terminal)
cd frontend
cp .env.example .env
npm install
npm run dev                 # http://localhost:5173
```

## 4. Configurar Stripe (cobro de $30/mes)

1. Crea una cuenta en https://dashboard.stripe.com
2. Crea un **producto** "InventPro Estándar" con un **precio recurrente mensual de USD 30**.
3. Copia el `price_id` a `STRIPE_PRICE_ID` en `backend/.env`.
4. Copia tu clave secreta a `STRIPE_SECRET_KEY`.
5. Crea un endpoint de webhook apuntando a `https://tu-dominio.com/api/billing/webhook`, escuchando:
   `checkout.session.completed`, `invoice.paid`, `invoice.payment_failed`, `customer.subscription.deleted`.
   Copia el secreto firmante a `STRIPE_WEBHOOK_SECRET`.

Con esto, cuando una empresa haga clic en "Activar suscripción", se le cobrarán $30 USD cada mes
automáticamente y el sistema activará o suspenderá su acceso según el estado del pago.

## 5. Despliegue a producción (recomendado)

Es un stack estándar, así que puedes desplegarlo en cualquiera de estas combinaciones:

| Componente | Opciones recomendadas |
|---|---|
| Backend (Node/Express) | Railway, Render, Fly.io, o un VPS con PM2 + Nginx |
| Base de datos PostgreSQL | Railway, Render, Supabase, Neon, o RDS |
| Frontend (estático) | Vercel, Netlify, Cloudflare Pages |

Pasos generales:
1. Despliega Postgres y copia su `DATABASE_URL`.
2. Despliega el backend, configura las variables de entorno (`.env`) y corre `npx prisma migrate deploy`.
3. Despliega el frontend con `VITE_API_URL` apuntando a la URL pública del backend.
4. Configura el webhook de Stripe con la URL pública de producción.
5. Agrega tu dominio propio (ej. `app.tuinventpro.com`) en el proveedor de hosting del frontend.

## 6. Próximos pasos sugeridos para robustecer el producto antes de vender a gran escala

- Reportes exportables a Excel/PDF (rotación de inventario, valorización, kardex por producto).
- Escaneo de código de barras desde cámara (móvil) para entradas/salidas rápidas.
- Notificaciones por correo/WhatsApp cuando un producto llega a stock mínimo.
- Facturación electrónica (integración con proveedor local según el país del cliente, ej. DIAN en Colombia).
- Roles y permisos más granulares (por módulo, no solo por rol global).
- App móvil (React Native) reutilizando la misma API.
- Panel de "superadmin" propio (para ti, el vendedor del SaaS) con métricas de todas las empresas clientas: MRR, churn, uso.

## 7. Estructura de carpetas relevante

```
backend/src/
├── index.ts                 # servidor Express, monta todas las rutas
├── middleware/auth.ts        # JWT + verificación de suscripción activa
├── middleware/errorHandler.ts
├── routes/auth.routes.ts     # registro (crea tenant) y login
├── routes/product.routes.ts
├── routes/catalog.routes.ts  # bodegas, categorías, proveedores, clientes
├── routes/stock.routes.ts    # ajustes, traslados, kardex
├── routes/purchaseOrder.routes.ts
├── routes/salesOrder.routes.ts
├── routes/dashboard.routes.ts
└── routes/billing.routes.ts  # Stripe checkout, portal, webhook
backend/prisma/schema.prisma  # modelo de datos completo

frontend/src/
├── context/AuthContext.tsx
├── components/Layout.tsx
└── pages/                    # Dashboard, Products, Stock, PurchaseOrders, SalesOrders, Suppliers, Customers, Billing, Login, Register
```
