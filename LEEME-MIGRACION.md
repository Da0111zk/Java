# Migración del proyecto Peluquería (JWT + MySQL + microservicios de la guía)

Este paquete **parte de tu repo `Peluqueria`**: los 5 servicios existentes (`cliente`, `profesional`, `tipoServicio`, `disponibilidadProfesional`, `agenda`), `eureka` y `gateway` se conservan con sus paquetes, entidades, DTOs, controladores, servicios, rutas, Lombok, Eureka y Swagger. Sobre eso se agregó la tecnología de 03_final (JWT, MySQL, Docker/.env) y los microservicios que faltaban según la guía 1.3.2.

> No se pudo compilar ni ejecutar en el entorno donde se generó (sin acceso a Maven Central). Hay que hacer `docker compose up --build` y `mvn test` y corregir lo que aparezca.

## Cómo ejecutar

```
./init-env.ps1            # Windows (PowerShell)   |   ./init-env.sh   # Linux/macOS
docker compose up -d --build
```

`init-env` crea `.env` con contraseñas y `JWT_SECRET` aleatorios y muestra la clave del usuario `admin`. Luego: `POST http://localhost:8080/auth/login` y usa el token como `Authorization: Bearer <token>`. Hay una colección Postman en `postman/` que recorre el flujo completo por el gateway.

## Servicios y rutas (todo entra por el gateway, puerto 8080)

| Servicio | Puerto | Ruta | Estado |
|---|---|---|---|
| tipoServicio | 8081 | /api/tiposervicio | existente |
| profesional | 8082 | /api/profesional | existente |
| disponibilidadProfesional | 8083 | /api/disponibilidad | existente |
| cliente | 8084 | /api/cliente | existente |
| agenda | 8085 | /api/agenda | existente |
| auth | 8086 | /auth | nuevo |
| registro | 8087 | /api/registro | nuevo |
| producto | 8088 | /api/producto | nuevo |
| bodega | 8089 | /api/bodega | nuevo |
| kardex | 8090 | /api/kardex | nuevo |
| boleta | 8091 | /api/boleta | nuevo |
| pago | 8092 | /api/pago | nuevo |
| notificacion | 8093 | /api/notificacion | nuevo |

Los nuevos siguen tu convención: nombres en singular, paquetes `Model/Repository/Service/Controller/Config/Exception/dto`, DTOs con `@Schema`, Lombok, `Optional` en los servicios, `DataInitializer` con datos de ejemplo, fechas como `String` (`yyyy-MM-dd`) y precios enteros (CLP).

## Seguridad

- `auth` emite JWT (HS256) con el rol (`ADMIN`, `RECEPCION`, `PROFESIONAL`) y vigencia de 15 minutos (`JWT_EXPIRATION_MS`).
- Cada servicio valida el token con el mismo `JWT_SECRET` (carpeta `Security/` y `Config/SecurityConfig`). Todo `/api/**` exige token y los `DELETE` exigen `ADMIN`. Swagger queda abierto y ahora tiene botón *Authorize*.
- Cuando un servicio llama a otro por WebClient reenvía el token de la petición (`TokenForwarder`).
- Ningún secreto está en el código: todo sale de `.env` (no se versiona).

## Cambios en archivos que ya existían (26)

Detalle línea por línea en `docs/cambios-archivos-existentes.diff`.

- **Los 5 servicios:** `pom.xml` (Oracle → `mysql-connector-j`, + Spring Security y jjwt), `application.yaml` (driver MySQL, sin dialecto Oracle, `ddl-auto: update`, `app.jwt.secret`), `Dockerfile` (se quitó el `COPY` del Wallet) y `SwaggerConfig` (esquema Bearer). No se tocaron entidades, DTOs, controladores ni repositorios, así que los tests existentes con constructores posicionales siguen válidos.
- **agenda y disponibilidadProfesional:** las llamadas WebClient reenvían el JWT. `agenda` además notifica al crear una cita y al cancelarla (falla sin romper la cita).
- **gateway:** rutas de los 8 servicios nuevos.
- **docker-compose.yaml:** reescrito con MySQL (un esquema y un usuario por servicio), los 13 servicios, Eureka y gateway.

## Errores del repo base que corregí porque bloqueaban el flujo

1. `agenda` consultaba `/api/tipoServicio/{id}` pero el controlador es `/api/tiposervicio` (las rutas distinguen mayúsculas) y además leía el JSON como `Long`.
2. `agenda` consultaba `/api/disponibilidadProfesional/...` (ruta inexistente) y esa consulta respondía 200 aunque no hubiera horario, así que nunca rechazaba. Agregué `GET /api/disponibilidad/profesional/{rut}/disponible/{fecha}/{hora}` y `agenda` ahora usa ese resultado.
3. `agenda.actualizar` siempre fallaba con «ya tiene una reserva» porque la cita chocaba consigo misma; ahora solo valida si cambia profesional, fecha u hora.
4. El gateway enrutaba `/api/tipoServicio/**` pero el controlador es en minúsculas.

## Cosas que vi y NO toqué

- `ProfesionalService.actualizar` no copia el campo `estado`.
- Las carpetas `Wallet_*` de Oracle están **versionadas en tu repo de GitHub**; quedan en el historial aunque se borren ahora. Conviene rotar esas credenciales. En este paquete no se incluyen.
- `eureka/application.yaml` exige `RENDER_EXTERNAL_HOSTNAME`; en el compose se define como `eureka` para ejecución local.
- Se mantuvo Spring Boot 4.0.7 y Spring Cloud 2025.1.2 (tus versiones) para no romper Eureka/Gateway; jjwt 0.13.0 es el de 03_final.
- El envío de notificaciones y el cobro de pagos son simulados.

Fichas de cada servicio según la guía: `docs/fichas-microservicios.md`.
