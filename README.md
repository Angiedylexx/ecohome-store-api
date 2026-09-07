# EcoHome Store — Migración del catálogo a PostgreSQL

Actividad 1: migración de la persistencia del catálogo de productos desde
un arreglo en memoria hacia PostgreSQL, con arquitectura MVC.

## Estructura del proyecto

```
unidad_01/
├── schema.sql                     # DDL de las tablas users y products
├── .env.example                   # Plantilla de variables de entorno
├── package.json
├── server.js                      # Punto de entrada: init() + listen()
├── src/
│   ├── app.js                     # Configuración de Express
│   ├── config/
│   │   └── db.js                  # Pool de conexiones (pg)
│   ├── models/
│   │   └── product.model.js       # init(), getAll(), create()
│   ├── controllers/
│   │   └── product.controller.js
│   └── routes/
│       └── product.routes.js
└── evidencias/
    └── evidencia-persistencia.md  # Evidencia real de la prueba de persistencia
```

## 1. Requisitos previos

- Node.js 18+
- Una instancia de PostgreSQL accesible (local, Docker, o en la nube)

## 2. Instalación

```bash
npm install
cp .env.example .env
# Edita .env con tus credenciales reales de PostgreSQL
```

## 3. Levantar PostgreSQL (opción rápida con Docker)

```bash
docker run -d --name ecohome_pg \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=postgres \
  -e POSTGRES_DB=ecohome_store \
  -p 5432:5432 postgres:16-alpine
```

Ajusta `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASS` y `DB_NAME` en `.env`
según corresponda.

## 4. Aplicar el esquema

```bash
psql -h localhost -U postgres -d ecohome_store -f schema.sql
# o, si usas el contenedor de Docker de arriba:
cat schema.sql | docker exec -i ecohome_pg psql -U postgres -d ecohome_store
```

> Nota: `product.model.js` también incluye un método `init()` que crea la
> tabla `products` automáticamente si no existe, por lo que este paso es
> redundante para esa tabla pero necesario para `users`.

## 5. Levantar el servidor Express

```bash
npm start
# Servidor EcoHome Store escuchando en http://localhost:4500 (o el PORT que definas)
```

## 6. Probar los endpoints

> **Nota para usuarios de PowerShell:** en PowerShell, `curl` es un alias
> de `Invoke-WebRequest`, que **no** acepta las banderas `-H`/`-d` al
> estilo de curl real. Usa `Invoke-RestMethod` (comandos de abajo), o si
> prefieres la sintaxis clásica de curl, invoca el binario real con
> `curl.exe` en vez de `curl`.

**Listar productos** (catálogo vacío al inicio):

```powershell
Invoke-RestMethod -Uri http://localhost:4500/products -Method Get
```

**Insertar un producto** (PowerShell, precios en pesos colombianos - COP):

```powershell
Invoke-RestMethod -Uri http://localhost:4500/products -Method Post `
  -ContentType "application/json" `
  -Body '{"name":"Bombilla LED Eco","price":35000}'

Invoke-RestMethod -Uri http://localhost:4500/products -Method Post `
  -ContentType "application/json" `
  -Body '{"name":"Panel Solar Portatil","price":450000}'
```

Equivalente con curl real (`curl.exe`) o en Bash/Linux/Mac:

```bash
curl.exe -X POST http://localhost:4500/products -H "Content-Type: application/json" -d '{\"name\":\"Bombilla LED Eco\",\"price\":35000}'
```

**Insertar un producto** (formato JSON para Postman, `POST /products`,
body raw/JSON):

```json
{
  "name": "Panel Solar Portatil",
  "price": 450000
}
```

**Volver a listar** para confirmar la inserción:

```powershell
Invoke-RestMethod -Uri http://localhost:4500/products -Method Get
```

## 7. Demostrar que la persistencia sobrevive a un reinicio

1. Inserta uno o más productos con `POST /products` como en el paso anterior.
2. Confirma que aparecen con `GET /products` (`Invoke-RestMethod ... -Method Get`).
3. Detén el servidor (`Ctrl + C` en la terminal donde corre `npm start`,
   o mata el proceso).
4. Verifica que el servidor ya no responde:
   ```powershell
   Invoke-RestMethod -Uri http://localhost:4500/products -Method Get
   # Error: "No se puede establecer conexión..." -> el proceso está apagado
   ```
5. Vuelve a levantar el servidor:
   ```bash
   npm start
   ```
6. Repite `GET /products`: **los mismos productos, con los mismos `id` y
   `created_at`, deben seguir ahí**. Esto demuestra que los datos ya no
   viven en la memoria del proceso Node (que se perdería en el paso 3),
   sino en PostgreSQL.

Como evidencia adicional, puedes consultar la base de datos directamente,
sin pasar por la API:

```bash
docker exec ecohome_pg psql -U postgres -d ecohome_store \
  -c "SELECT * FROM products ORDER BY id;"
```

Esta prueba completa ya se ejecutó de principio a fin durante el
desarrollo de esta actividad; el detalle con las salidas reales está en
[`evidencias/evidencia-persistencia.md`](./evidencias/evidencia-persistencia.md),
incluyendo además una prueba de resistencia a inyección SQL sobre el
endpoint `POST /products`.
