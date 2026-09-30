# Sistema de Turnos Médicos (Django)

Aplicación web para gestionar médicos, pacientes y turnos (citas) médicas.

## Funcionalidades

- **Especialidades y médicos**: cada médico tiene una especialidad, jornada laboral
  (hora de inicio/fin) y duración de turno configurable.
- **Pacientes**: registro simple con RUT/documento único.
- **Turnos**:
  - Agendar, ver detalle, cambiar estado (Pendiente, Confirmado, Cancelado, Atendido, No asistió) y cancelar.
  - Validaciones automáticas: no se permite agendar en el pasado, fuera del horario
    del médico, ni dos turnos en el mismo horario con el mismo médico (a nivel de
    base de datos con `UniqueConstraint` + validación en `clean()`).
  - Endpoint JSON (`/api/horarios-disponibles/`) que calcula los horarios libres
    de un médico en una fecha, usado por el formulario de agendamiento.
- **Panel de administración** de Django ya configurado (`/admin/`) para gestión
  interna rápida.
- Interfaz con Bootstrap 5, en español.

## Estructura del proyecto

```
turnos_medicos/
├── manage.py
├── requirements.txt
├── config/              # Configuración del proyecto (settings, urls)
└── citas/                # App principal
    ├── models.py         # Especialidad, Medico, Paciente, Turno
    ├── admin.py
    ├── forms.py
    ├── views.py
    ├── urls.py
    ├── templates/citas/
    ├── static/citas/
    └── management/commands/cargar_datos_demo.py
```

## Instalación

1. Crea y activa el ambiente virtual (PowerShell):
   ```
   python -m venv .venv
   .\.venv\Scripts\Activate
   python -m pip install --upgrade pip
   ```
   Si Windows bloquea el script: `Set-ExecutionPolicy Bypass -Scope CurrentUser`.

2. Instala las librerías:
   ```
   pip install -r requirements.txt
   ```

3. Crea la base de datos y el usuario en MySQL ejecutando `docs/crear_base_datos.sql`
   como administrador (`mysql -u root -p`).

4. Revisa el archivo `.env` (copia de `.env.example`): `SECRET_KEY` y las credenciales
   `DB_*` deben coincidir con las del paso 3. El `.env` no se sube al repositorio.

5. Crea las tablas (las migraciones ya vienen incluidas):
   ```
   python manage.py migrate
   ```

6. Crea el superusuario y carga datos de ejemplo:
   ```
   python manage.py createsuperuser
   python manage.py cargar_datos_demo
   ```

7. Inicia el servidor. Con `DEBUG=False` Django no sirve el CSS, por eso en local se usa `--insecure`:
   ```
   python manage.py runserver --insecure
   ```
   App: http://127.0.0.1:8000/ · Admin: http://127.0.0.1:8000/admin/

Cada vez que cambies los modelos: `makemigrations` y `migrate`.
Cada vez que agregues una librería: `pip freeze > requirements.txt`.

## Flujo de uso típico

1. Entra a `/admin/` y crea (o usa el comando `cargar_datos_demo`) especialidades y médicos.
2. Registra un paciente desde "Nuevo paciente" en el menú.
3. Ve a "Agendar turno", elige paciente, médico, fecha y hora (el sistema te
   muestra los horarios libres del médico ese día).
4. Desde "Turnos" puedes filtrar por estado, médico o fecha, ver el detalle,
   cambiar su estado clínico o cancelarlo.

## Flujo URL → vista → plantilla

Django resuelve cada petición en tres pasos, y este proyecto los usa así:

1. **`config/urls.py`** (raíz del proyecto) incluye las rutas de la app con
   `include('citas.urls')`, y define `handler404` / `handler500` para los
   errores.
2. **`citas/urls.py`** mapea cada ruta a una vista concreta, por ejemplo
   `path('', views.HomeView.as_view(), name='home')` es la página de bienvenida.
3. La **vista** (en `citas/views.py`) procesa la lógica (consultas al ORM,
   validaciones, formularios) y llama a `render()` con una **plantilla** de
   `citas/templates/citas/`, que hereda de `base.html`.

### Página de bienvenida

La ruta raíz (`/`) apunta a `HomeView`, que muestra un resumen (médicos
activos, pacientes registrados, turnos de hoy) en vez de la página de
bienvenida por defecto de Django — confirma que la app está correctamente
conectada al proyecto.

### Página 404 personalizada

- `config/urls.py` define `handler404 = 'citas.views.error_404_view'`.
- `citas/views.py` → `error_404_view()` renderiza `citas/templates/citas/404.html`
  (hereda el diseño del sitio) devolviendo explícitamente `status=404`.
- También se agregó `handler500` con `citas/templates/citas/500.html` como
  buena práctica adicional.

**Cómo probarla:** Django solo usa `handler404`/`handler500` cuando
`DEBUG = False` (con `DEBUG = True` siempre verás la página de depuración de
Django con el traceback). Para probarla localmente:

```bash
# En config/settings.py, cambia temporalmente:
DEBUG = False

# Luego levanta el servidor y visita una URL que no existe, por ejemplo:
python manage.py runserver
# http://127.0.0.1:8000/esta-ruta-no-existe/
```

Verás la plantilla personalizada con el mensaje "😕 No encontramos la página
que buscas" en vez del error técnico de Django. No olvides volver a poner
`DEBUG = True` para seguir desarrollando.

## Próximos pasos sugeridos (no incluidos por simplicidad)

- Autenticación de pacientes/médicos con login propio y permisos por rol.
- Notificaciones por correo/SMS al confirmar o cancelar un turno.
- Recordatorios automáticos (Celery + cron).
- API REST (Django REST Framework) para integrarlo con una app móvil.

## Notas técnicas

- Base de datos por defecto: SQLite (`db.sqlite3`), ideal para desarrollo. Para
  producción, cambia `DATABASES` en `config/settings.py` a PostgreSQL/MySQL.
- Recuerda cambiar `SECRET_KEY` y poner `DEBUG = False` con un `ALLOWED_HOSTS`
  correcto antes de desplegar en producción.
