# TeleConnect - Plataforma de Videoconferencias en Django (Estilo Google Meet)

Aplicación web integral para videoconferencias, gestión de sesiones virtuales, control de conexiones/periféricos e integración con servidor seguro por medio de VPN.

---

## 🚀 Características Principales

1. **Interfaz Inspirada en Google Meet**:
   - Cabecera con selector de código/vínculo y botón directo **"Unirse"**.
   - Botón **"+ Nuevo"** con opciones para:
     - Crear una reunión para más tarde (con enlace copiable).
     - Iniciar una reunión instantánea.
     - Programar en calendario.
   - Navegación lateral entre **"Reuniones"** y **"Llamadas"**.
   - **Selector Semanal Interactivo**: Navegación día por día con fecha resaltada y carrusel semanal (`< LUN, MAR, MIÉ, JUE, VIE, SÁB, DOM >`).
   - Pantalla de **estado vacío ilustrada** cuando no hay reuniones para el día seleccionado.

2. **Gestión Completa de Periféricos (Audio y Video)**:
   - **Modal de Configuración**:
     - **Pestaña Audio**: Selector de micrófono, selector de bocina/salida de audio, medidor dinámico de decibelios de voz (vúmetro animado) y botón "Probar" de bocina que reproduce tono armónico.
     - **Pestaña Video**: Selector de cámara web con previsualización en vivo en tiempo real.
   - Detección y enumeración automática de hardware mediante la API `navigator.mediaDevices`.

3. **Sala de Videoconferencia**:
   - **Sala de Preparación (Pre-join / Lobby)**: Vista previa de cámara y micrófono antes de ingresar a la llamada.
   - **Sala Activa**:
     - Cuadrícula responsiva de video (video local y participantes).
     - Barra de herramientas inferior estilo Google Meet:
       - Activar / silenciar micrófono.
       - Activar / desactivar cámara web.
       - Compartir pantalla completa (`getDisplayMedia`).
       - Levantar la mano.
       - Chat interno de la llamada.
       - Panel de participantes.
       - Abandonar / colgar llamada.

4. **Gestión de Admisión ("Aceptar Conexiones")**:
   - Sala de espera para invitados.
   - Alertas emergentes (Toast) para el anfitrión: *"{Nombre} quiere unirse a esta llamada"*.
   - Botones para **"Permitir"** o **"Denegar"** el acceso en tiempo real.

5. **Servidor Seguro y Conexión por VPN**:
   - Monitoreo en tiempo real del estado del túnel seguro de la VPN.
   - Verificación de subred (`10.8.0.x`) y cifrado seguro (TLS 1.3 / OpenVPN).
   - Indicador visual permanente en el encabezado y en la sala de llamadas.

---

## 🛠️ Requisitos e Instalación para el Equipo

1. **Instalar dependencias**:
   ```bash
   pip install -r requirements.txt
   ```

2. **Configurar variables locales (opcional)**:
   Copia la plantilla de entorno para ajustar tus parámetros locales sin subir claves al repositorio:
   ```bash
   cp .env.example .env
   ```

3. **Aplicar migraciones**:
   Esto creará automáticamente la base de datos local SQLite de forma aislada en tu equipo:
   ```bash
   python manage.py makemigrations
   python manage.py migrate
   ```

4. **Iniciar el servidor**:
   ```bash
   python manage.py runserver
   ```
   Accede en tu navegador a: `http://localhost:8000/`

---

## 🔒 Seguridad y Colaboración en Git

- **Sin claves ni bases de datos en Git:** La base de datos local (`db.sqlite3`), los registros y los archivos de secretos (`.env`) están estrictamente excluidos en `.gitignore`.
- **Cuentas aisladas:** Cada colaborador que clone el repositorio tendrá su propia base de datos local independiente; las cuentas y contraseñas nunca se comparten por el repositorio de código.
- **Crear tu usuario local:** Puedes registrarte desde la pantalla de inicio o crear un administrador con:
  ```bash
  python manage.py createsuperuser
  ```

---

## 🌐 Despliegue en Producción y Servidor VPN

Para poner la plataforma en un servidor real con dominio propio, certificado SSL/HTTPS (necesario para permitir cámara y micrófono en conexiones remotas) y configurar el túnel seguro VPN, consulta la guía completa en:

📖 **[DEPLOYMENT.md](DEPLOYMENT.md)**

