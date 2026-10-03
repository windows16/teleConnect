# TeleConnect - Plataforma de Videoconferencias en Django

TeleConnect es una plataforma web de videoconferencias inspirada en Google Meet. El proyecto combina un backend Django monolítico modular, persistencia relacional, autenticación local o mediante Supabase, APIs JSON para la operación de las salas y una capa de verificación de conectividad segura mediante VPN.

> **Estado actual:** la comunicación multimedia entre participantes se apoya en WebRTC en el navegador. Django actúa como backend de aplicación, autorización, persistencia, admisión, chat, presencia y señalización básica; no funciona como servidor multimedia SFU/MCU.

---

## 🏗️ Arquitectura del backend

### Vista general

El backend está organizado como un proyecto Django con una aplicación de dominio principal:

```text
Cliente web
    │
    ├── Plantillas Django + JavaScript
    │       ├── WebRTC y dispositivos multimedia
    │       ├── Polling de admisión, chat y presencia
    │       └── Consumo de endpoints JSON
    │
    ▼
Django / WSGI / ASGI
    │
    ├── core/
    │   ├── settings.py       Configuración, seguridad y servicios externos
    │   ├── urls.py            Enrutamiento raíz
    │   ├── wsgi.py            Entrada para servidores WSGI/Gunicorn
    │   └── asgi.py            Entrada ASGI
    │
    └── meetings/
        ├── urls.py            Rutas HTML y API
        ├── views.py           Casos de uso y controladores HTTP
        ├── models.py          Entidades y persistencia
        ├── forms.py           Validación de formularios
        ├── supabase_auth.py   Integración opcional de autenticación
        └── vpn_service.py     Verificación del canal seguro
    │
    ▼
Base de datos SQLite local
    │
    └── Opcionalmente, servicios externos
        └── Supabase Auth
```

### Capas y responsabilidades

#### 1. Configuración y arranque: `core/`

- `core/settings.py` centraliza la configuración del proyecto, middleware, aplicaciones instaladas, plantillas, base de datos, archivos estáticos, zona horaria y variables de entorno.
- `core/urls.py` expone el panel administrativo y delega las rutas de la aplicación `meetings`.
- `core/wsgi.py` permite desplegar la aplicación con servidores WSGI como Gunicorn.
- `core/asgi.py` proporciona el punto de entrada ASGI para escenarios compatibles con ejecución asíncrona.
- WhiteNoise permite servir archivos estáticos directamente desde la aplicación en despliegues sencillos.

#### 2. Módulo de dominio: `meetings/`

La aplicación `meetings` concentra las capacidades funcionales de la plataforma:

- autenticación y gestión de sesión;
- creación, consulta y programación de reuniones;
- lobby y admisión de participantes;
- chat persistente por sala;
- presencia y estado de participantes;
- diagnóstico de conectividad VPN;
- renderizado de las vistas HTML y exposición de endpoints JSON.

Actualmente, los controladores se encuentran en `meetings/views.py`. Estos controladores validan la sesión del usuario, ejecutan las operaciones de dominio mediante el ORM de Django y devuelven plantillas HTML o respuestas `JsonResponse`.

#### 3. Persistencia y modelo de datos

La aplicación usa el ORM de Django sobre SQLite para el entorno local. Las principales entidades son:

- **`User`**: usuario proporcionado por el sistema de autenticación de Django.
- **`Meeting`**: sesión de videoconferencia, código de acceso, anfitrión, agenda, estado y requisito de VPN.
- **`ConnectionRequest`**: solicitud de entrada de un participante y su estado `PENDING`, `ACCEPTED` o `REJECTED`.
- **`CallContact`**: contactos disponibles en la sección de llamadas.
- **`ChatMessage`**: mensajes persistentes asociados a una reunión.
- **`RoomParticipant`**: presencia temporal en una sala, `peer_id`, estado de audio/video y última actividad.

Las relaciones principales son:

```text
User 1 ─── N Meeting                 (anfitrión)
User 1 ─── N CallContact              (contactos)
Meeting 1 ─── N ConnectionRequest     (admisión)
Meeting 1 ─── N ChatMessage            (chat)
Meeting 1 ─── N RoomParticipant        (presencia)
User 1 ─── N ChatMessage
User 1 ─── N RoomParticipant
```

Las migraciones se encuentran en `meetings/migrations/` y deben aplicarse antes de iniciar la aplicación.

#### 4. Autenticación y sesiones

TeleConnect soporta dos modos de autenticación:

- **Django local:** utiliza `django.contrib.auth` para registrar, autenticar y cerrar sesiones.
- **Supabase Auth:** se activa cuando existen `SUPABASE_URL` y `SUPABASE_ANON_KEY`. El servicio `meetings/supabase_auth.py` delega el registro e inicio de sesión a Supabase y sincroniza el usuario remoto con un usuario local de Django.

La sincronización local permite que las reuniones, los mensajes, los contactos y los permisos sigan utilizando las relaciones estándar del ORM de Django. Los tokens de Supabase se almacenan en la sesión Django para mantener el contexto de autenticación durante la navegación.

Las vistas protegidas usan `@login_required`, y las operaciones sensibles de anfitrión verifican explícitamente que el usuario autenticado sea el propietario de la reunión.

#### 5. API JSON y flujo de una reunión

Las rutas de la API se definen en `meetings/urls.py` y son atendidas por funciones en `meetings/views.py`:

| Área | Endpoints principales | Responsabilidad |
| --- | --- | --- |
| Reuniones | `/api/meetings/instant/`, `/api/meetings/schedule/` | Crear reuniones instantáneas o programadas. |
| Consulta | `/api/meetings/check/` | Validar un código de reunión y generar enlaces de acceso. |
| Admisión | `/api/waiting-room/<code>/status/` | Consultar el estado del participante. |
| Admisión | `/api/waiting-room/<code>/pending/` | Listar solicitudes pendientes para el anfitrión. |
| Admisión | `/api/waiting-room/<code>/action/` | Aceptar o rechazar una solicitud. |
| VPN | `/api/vpn/status/` | Obtener el estado y diagnóstico de la conexión segura. |
| Chat | `/api/room/<code>/messages/` | Leer mensajes o publicar un nuevo mensaje. |
| Presencia | `/api/room/<code>/heartbeat/` | Registrar actividad y devolver participantes activos. |
| Presencia | `/api/room/<code>/leave/` | Eliminar la presencia al abandonar la sala. |

Flujo principal de acceso:

1. El usuario crea o busca una reunión mediante la interfaz.
2. Django genera o valida un código único de reunión.
3. La vista `prejoin` carga la reunión y el estado de VPN.
4. El usuario entra a `room/<code>/`.
5. Si no es el anfitrión, se crea o recupera una `ConnectionRequest`.
6. El anfitrión consulta las solicitudes pendientes y decide aceptar o rechazar.
7. Los participantes envían heartbeats periódicos; se consideran activos durante los últimos 20 segundos.
8. El chat se persiste en `ChatMessage` y puede consultarse incrementalmente mediante `after_id`.
9. El navegador establece las conexiones WebRTC y Django mantiene el estado de coordinación de la sala.

#### 6. Presencia y señalización básica

`RoomParticipant` funciona como registro de presencia temporal. Cada heartbeat actualiza:

- identificador `peer_id` utilizado por el cliente WebRTC;
- nombre visible y rol de anfitrión;
- estado de micrófono y cámara;
- estado de mano levantada;
- fecha de última actividad.

El backend devuelve los participantes cuya actividad ocurrió dentro de una ventana de 20 segundos. Esta estrategia permite detectar desconexiones sin requerir un canal WebSocket, aunque el frontend debe continuar enviando heartbeats mientras el usuario permanezca en la sala.

> La versión actual utiliza endpoints HTTP y consultas periódicas. Para una evolución hacia tiempo real de mayor escala, se puede incorporar Django Channels, Redis y WebSockets.

#### 7. Capa de conectividad segura: `vpn_service.py`

El servicio VPN obtiene la IP del cliente desde `X-Forwarded-For` o `REMOTE_ADDR` y la compara con la subred configurada. Devuelve un objeto de diagnóstico con:

- IP del cliente y del servidor;
- nombre del servidor seguro;
- prefijo de subred permitido;
- estado de conexión;
- etiqueta y clase visual para la interfaz;
- protocolo declarado para el canal seguro.

El contexto `vpn_context_processor` inyecta esta información en todas las plantillas. Además, las vistas de pre-join, sala y API de VPN consultan el estado de forma explícita.

Las variables principales son `VPN_SERVER_IP`, `VPN_SUBNET_PREFIX`, `VPN_SERVER_PORT`, `VPN_SERVER_NAME` y `VPN_ENFORCE`.

> La validación actual se basa en la IP observada por Django y en el modo local. La aplicación no establece por sí misma un túnel OpenVPN ni valida criptográficamente el cifrado; esa responsabilidad corresponde a la infraestructura de red y al proxy/VPN desplegado.

---

## 🚀 Funcionalidades

- Interfaz de reuniones inspirada en Google Meet.
- Creación de reuniones instantáneas y programadas.
- Códigos de reunión únicos y enlaces de pre-join.
- Lobby con aceptación o rechazo por parte del anfitrión.
- Chat persistente dentro de la sala.
- Presencia de participantes mediante heartbeat.
- Vista previa y control de cámara, micrófono y pantalla mediante APIs del navegador.
- Comunicación de audio/video mediante WebRTC en el cliente.
- Monitoreo de conectividad segura y VPN.
- Autenticación local con Django o autenticación externa con Supabase.

---

## 📁 Estructura del proyecto

```text
.
├── core/
│   ├── settings.py
│   ├── urls.py
│   ├── asgi.py
│   └── wsgi.py
├── meetings/
│   ├── migrations/
│   ├── forms.py
│   ├── models.py
│   ├── supabase_auth.py
│   ├── urls.py
│   ├── views.py
│   └── vpn_service.py
├── static/
│   ├── css/
│   └── js/
├── templates/
│   ├── auth/
│   ├── components/
│   └── meetings/
├── manage.py
├── requirements.txt
├── .env.example
└── DEPLOYMENT.md
```

---

## 🛠️ Requisitos e instalación

1. Instalar dependencias:

   ```bash
   pip install -r requirements.txt
   ```

2. Configurar variables locales opcionales:

   ```bash
   cp .env.example .env
   ```

3. Aplicar migraciones:

   ```bash
   python manage.py makemigrations
   python manage.py migrate
   ```

4. Crear un administrador local si es necesario:

   ```bash
   python manage.py createsuperuser
   ```

5. Iniciar el servidor de desarrollo:

   ```bash
   python manage.py runserver
   ```

   Accede en el navegador a `http://localhost:8000/`.

---

## ⚙️ Configuración del backend

Las variables se cargan desde `.env` o desde el entorno del proceso. Las más importantes son:

```env
DJANGO_SECRET_KEY=cambia-esta-clave-en-produccion
DJANGO_DEBUG=True
DJANGO_ALLOWED_HOSTS=localhost,127.0.0.1
DJANGO_CSRF_TRUSTED_ORIGINS=http://localhost:8000

SUPABASE_URL=
SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=

VPN_SERVER_IP=10.8.0.1
VPN_SUBNET_PREFIX=10.8.0.
VPN_SERVER_PORT=443
VPN_SERVER_NAME=Telecom-Secure-Server
VPN_ENFORCE=False
```

En producción se recomienda:

- establecer `DJANGO_DEBUG=False`;
- utilizar una `DJANGO_SECRET_KEY` segura y externa al repositorio;
- restringir `DJANGO_ALLOWED_HOSTS`;
- configurar correctamente `DJANGO_CSRF_TRUSTED_ORIGINS`;
- proteger las claves de Supabase mediante secretos del proveedor de despliegue;
- sustituir SQLite por una base de datos gestionada cuando aumente la concurrencia;
- usar HTTPS para habilitar cámara, micrófono y pantalla en navegadores remotos.

---

## 🔒 Seguridad y colaboración

- La autenticación de vistas se controla con sesiones Django y `@login_required`.
- Las acciones de admisión verifican que el usuario sea el anfitrión de la reunión.
- Los endpoints mutables deben utilizar el mecanismo CSRF de Django desde el frontend.
- `db.sqlite3`, `.env`, registros y secretos no deben versionarse.
- La base de datos local es independiente para cada colaborador.
- La capa VPN expone indicadores de estado, pero la seguridad real del túnel debe configurarse en la infraestructura.

---

## 🧪 Comandos útiles

```bash
# Verificar la configuración del proyecto
python manage.py check

# Ejecutar pruebas
python manage.py test

# Crear nuevas migraciones después de modificar modelos
python manage.py makemigrations

# Aplicar migraciones
python manage.py migrate

# Recolectar archivos estáticos para producción
python manage.py collectstatic
```

---

## 🌐 Despliegue

Para desplegar TeleConnect con Gunicorn, HTTPS, archivos estáticos y configuración de VPN, consulta:

📖 **[DEPLOYMENT.md](DEPLOYMENT.md)**

---

## 🔭 Evolución recomendada de la arquitectura

Para escalar el backend y mejorar el tiempo real, las siguientes mejoras son candidatas naturales:

1. Separar servicios y casos de uso de `views.py` en módulos como `services/`, `selectors/` y `api/`.
2. Incorporar Django REST Framework para serialización, validación y versionado formal de la API.
3. Migrar la presencia, admisión y chat de polling HTTP a Django Channels/WebSockets.
4. Utilizar Redis para presencia efímera, publicación de eventos y coordinación entre procesos.
5. Migrar de SQLite a PostgreSQL en entornos compartidos o de producción.
6. Añadir pruebas unitarias y de integración para autenticación, permisos, admisión, chat y VPN.
7. Incorporar un proveedor SFU/MCU si se requiere administrar el tráfico multimedia desde el servidor y soportar salas de mayor tamaño.
