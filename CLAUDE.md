# Plataforma centralizada de torneos y gráficas en vivo

## Visión general
Plataforma web multi-cliente con dos módulos sobre una misma base:
1. **Gestión de torneos**: inscripciones, fixture, resultados, varios torneos en simultáneo.
2. **Gráficas en vivo (broadcast graphics)**: widgets con URL para usar como browser source en OBS/vMix, controlados en tiempo real desde botoneras. Estilo Singular.live / H2R Graphics.

Los torneos alimentan a las gráficas con datos reales, pero el módulo de gráficas debe poder venderse solo, a clientes sin torneos (por ejemplo, streams de Kick).

Idioma de la interfaz: español.

## Entorno y despliegue
- Desarrollo en PC local, luego migración a un VPS.
- **Docker desde el día uno**: lo que corre local es lo mismo que corre en el VPS.
- Repositorio **privado** en GitHub. Monorepo con carpetas separadas por módulo.
- Tres ambientes: desarrollo (PC), staging (VPS) y producción (VPS), cada uno asociado a una rama.
- Despliegue automático con GitHub Actions. Producción protegida: nada entra sin pasar por staging.
- Base de datos: MySQL. Redis para pub/sub en tiempo real y sesiones.
- Archivos de clientes (imágenes, videos) fuera del repo, en almacenamiento portable preparado para migrar a almacenamiento de objetos.
- **Nunca** subir credenciales al repo: usar `.env` excluido en `.gitignore` y GitHub Secrets.
- Backups automáticos de la base de datos y los archivos.
- Pruebas externas mientras corre en la PC: túnel tipo Cloudflare Tunnel, sin abrir puertos.
- Los eventos en vivo reales corren desde el VPS, no desde la PC.

## Estructura de clientes y permisos
- Jerarquía: **Cliente → Proyectos/Competencias → Widgets y Botoneras**.
- Un usuario puede pertenecer a varios clientes, con un rol distinto en cada uno.
- Permisos granulares (RBAC con alcance por recurso) definidos por el super admin, no roles fijos.
- Roles de referencia:
  - **Super admin**: administra todo el sistema.
  - **Admin de cliente**: gestiona su propio equipo.
  - **Diseñador**: crea y edita widgets, no los opera en vivo.
  - **Operador**: solo accede a las botoneras asignadas.
  - **Productor o moderador de torneo**: carga resultados.
  - **Jugador o participante**: ve solo su información.
- Paneles: administrador global, panel de cliente y panel de usuario.

## Autenticación
- Login con Google (OAuth) y con mail y contraseña. Ambos métodos se vinculan a la **misma cuenta** si el mail coincide y está verificado.
- Personal de clientes entra por **invitación**, no por registro abierto.
- Verificación de mail, recuperación de contraseña y 2FA obligatorio para roles admin.
- OAuth en producción requiere dominio con HTTPS: definir el dominio temprano.

## Tiempo real (núcleo del sistema)
- El servidor es la **única fuente de verdad** del estado de cada widget.
- Flujo: botón → servidor actualiza estado → broadcast por WebSocket a los widgets suscritos.
- **Recuperación del estado**: si OBS recarga o se corta la conexión, el widget reconecta y restaura el estado actual sin repetir animaciones de entrada.
- **Indicador "al aire" (tally)** en la botonera, con confirmación de recepción por parte del widget.
- Vista previa y programa: ensayar una gráfica antes de mandarla al aire.
- URLs de widgets con token secreto revocable.
- Registro de acciones: quién ejecutó qué acción, cuándo y desde qué IP.

## Widgets
- Lienzo de resolución fija (1920×1080 y vertical 1080×1920) que escala. Probar en Chromium, que es el motor de OBS.
- Capas sin límite rígido (mínimo 20), con aviso de rendimiento.
- Todas las capas soportan transparencia y animación opcional.
- Tipos de capa:
  - texto;
  - imagen;
  - video con canal alfa (WebM);
  - formas;
  - contadores y temporizadores;
  - **capas de datos vinculados** a la base de datos (jugadores, marcadores, llaves del torneo), que se actualizan solas al cargar resultados.
- **Estados o escenas por widget** (ej.: "presentación", "marcador", "ganador") que definen qué capas entran y con qué animación.
- Versionado de widgets, duplicado y plantillas reutilizables entre clientes.

## Animaciones
- Sistema combinable, no una lista cerrada. Unos 15 tipos base (fade, slide, zoom, rotación, desenfoque, barrido con máscara, máquina de escribir, rebote, etc.) combinados con dirección, curva de aceleración, duración y retardo. Objetivo: más de 100 combinaciones.
- Presets guardables con nombre.
- Por capa se configuran:
  - animación de entrada (**in**) y de salida (**out**), cada una con su tiempo;
  - animación en bucle opcional mientras la capa está visible;
  - escalonado entre capas.
- Línea de tiempo por widget.

## Botoneras
- Configurables por cliente y asignables a operadores específicos.
- Acciones por botón:
  - mostrar u ocultar;
  - activar un estado;
  - cambiar un texto;
  - sumar o restar a un contador;
  - **macros** (secuencia de varias acciones).
- Atajos de teclado.
- API/webhooks para disparar acciones desde afuera (integración con Stream Deck vía Companion).

## Fases de desarrollo
1. **Base**: Docker, repo, login (Google + mail), clientes, usuarios y permisos.
2. **Circuito completo mínimo**: editor simple de widgets (pocas capas y animaciones), botonera y tiempo real de punta a punta, con recuperación del estado. Probar en un stream real.
3. **Ampliación**: biblioteca de animaciones combinable, estados, línea de tiempo, versionado.
4. **Torneos**: gestión de competencias y capas de datos vinculados.
5. **Integraciones**: API, webhooks, Stream Deck, registro de acciones avanzado.

## Decisiones pendientes
- Stack tecnológico: proponerlo con justificación antes de empezar.
- Definir si el módulo de torneos integra o reemplaza la plataforma existente del torneo de FIFA de la LPF (inscripción de socios de los 30 clubes, validación contra Excel del club, autorización de menores en PDF, notificaciones por WhatsApp, registro de IP por visita).
- Dominio para producción.

## Reglas de trabajo
- Antes de escribir código de una fase, proponer el plan y esperar aprobación.
- Commits chicos y descriptivos, en ramas por funcionalidad.
- No desplegar a producción en días de evento en vivo.
