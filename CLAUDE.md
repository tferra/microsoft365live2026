# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

# Adopción de Copilot con Microsoft Fabric

## Qué es este proyecto
Demo de la sesión "¿Está funcionando Copilot en tu empresa? Mide la adopción con Microsoft Fabric" (Microsoft 365 Live 2026, online, 4 de noviembre de 2026). Speaker: Toni Ferrà (Data Tech Lead, Bravent).

Un notebook descarga cada día los informes de uso de Microsoft 365 Copilot desde Microsoft Graph y guarda una foto diaria en un Lakehouse de Fabric. Sobre ese histórico se construye un modelo semántico Direct Lake y un informe de Power BI de adopción.

Mensaje clave: la API de Graph solo da la foto del periodo; el valor está en guardar la película (histórico diario).

## Estado actual
- [x] Notebook de ingesta `nb_ingesta_copilot_usage.ipynb`. Probado en local en modo simulado.
- [ ] Modelo semántico Import (TMDL) `sm_copilot_adoption`
- [ ] Informe Power BI (PBIR) "Adopción de Copilot"
- [ ] Tema JSON del informe

Lo pendiente se entrega como proyecto PBIP (modelo en TMDL, informe en PBIR).

## Estructura del repositorio (objetivo)
```
/nb_ingesta_copilot_usage.ipynb             # ingesta Graph -> Lakehouse (existe; hoy en la raíz, no en /notebooks)
/pbip/CopilotAdoption.pbip                  # por crear
/pbip/CopilotAdoption.SemanticModel/        # TMDL, Import
/pbip/CopilotAdoption.Report/               # PBIR + StaticResources/tema.json
/CLAUDE.md
```

## Entorno Fabric
| Elemento | Nombre |
|---|---|
| Workspace | M365Live - Adopción Copilot |
| Capacidad | F (de pago, necesaria para Copilot en Power BI) |
| Lakehouse | LH_Copilot (sin esquemas) |
| Notebook | nb_ingesta_copilot_usage (Lakehouse predeterminado adjunto) |
| Modelo semántico | CopilotAdoption (Import desde el endpoint SQL, conexión en la nube SQL_M365 con OAuth) |
| Informe | CopilotAdoption |
| Data Agent | da_copilot_adoption (sobre el modelo CopilotAdoption; preguntas de ejemplo en preguntas-data-agent.md) |
| Programación | Notebook diario a las 7:00, hora de Madrid, con MODO = "real" |

Confirma siempre los nombres reales con la conexión a Fabric antes de asumirlos.

## Autenticación y secretos
- App registration en Entra ID `fabric-copilot-adoption`, con permisos de aplicación de Graph Reports.Read.All y User.Read.All y consentimiento de administrador.
- Secretos en Key Vault, leídos con `notebookutils.credentials.getSecret`: `copilot-tenant-id`, `copilot-client-id`, `copilot-client-secret`.
- No escribas tenant IDs ni URLs reales de Key Vault en el repositorio. En el notebook `KEY_VAULT_URL` es el marcador `https://<tu-keyvault>.vault.azure.net/`.
- En el centro de administración de M365 hay que desactivar los nombres ocultos en los informes. Si no, los UPN llegan como hashes y el cruce con dim_users falla.

## Notebook de ingesta
Parámetros (celda con la etiqueta `parameters`, sobrescribibles desde un pipeline): `MODO` ("simulado" | "real"), `KEY_VAULT_URL` y los nombres de los tres secretos, `PERIODO = "D30"`, `DIAS_HISTORICO_SIM = 84`, `USUARIOS_SIM = 120`, `SEMILLA = 42`, `DOMINIO_SIM = "contoso.com"`.

Flujo:
1. Token client credentials contra Entra ID (solo en modo real).
2. Graph con paginación (`@odata.nextLink`) y reintento ante 429/503 (`graph_get`, `graph_get_all`):
   - Uso por usuario: JSON por `beta/copilot/reports/getMicrosoft365CopilotUsageUserDetail(period='D30')`; si falla, CSV por `v1.0/...` con cabeceras normalizadas a camelCase (`a_camel`).
   - Usuarios: `v1.0/users` con userPrincipalName, displayName, department, country, jobTitle, accountEnabled.
3. Modo real: guarda el bruto en `Files/raw/copilot_usage/snapshot_date=YYYY-MM-DD/usage.json`.
4. `normalizar_uso` produce la tabla por usuario y un unpivot de todas las columnas `*LastActivityDate` (salvo `lastActivityDate`) a usuario x app. `APP_MAP` traduce a nombres legibles; las apps nuevas que añada Microsoft entran con el nombre de columna sin el sufijo.
5. Escritura Delta (`escribir_hechos`, `escribir_dim`):
   - Hechos en modo real: `DELETE` de la foto de hoy y `append` (idempotente).
   - Hechos en modo simulado: sobrescribe todo.
   - Dimensiones: siempre se sobrescriben.
6. Comprobación SQL final: licencias, activos 28d y % de adopción de las últimas 14 fotos.

El modo simulado (`generar_simulado`) produce datos con la misma estructura que Graph, de modo que el resto del pipeline es idéntico en los dos modos:
- 120 usuarios repartidos en 6 departamentos (IT, Marketing, Comercial, Finanzas, Operaciones, RRHH) con adopción inicial y final por departamento en `DEPARTAMENTOS`.
- 84 fotos diarias, 2 días de retraso entre foto y refresco, 4 semanas de calentamiento.
- Un 10 % de licencias nunca usadas y un 15 % de usuarios que lo prueban y lo dejan.
- Uso por app según `PESO_APP` (Teams y Outlook los más usados; Loop y OneNote los menos).
- Las fechas son relativas a `HOY`: los números exactos cambian según el día de ejecución aunque la semilla sea fija.

Si cambias el esquema de alguna tabla del notebook, actualiza también el modelo y este documento.

## Tablas Delta (LH_Copilot)

**copilot_usage_user** · hechos · usuario x foto
| Columna | Tipo | Notas |
|---|---|---|
| snapshot_date | date | Día de la carga |
| report_refresh_date | date | Refresco del informe en Graph (2-3 días antes) |
| upn | string | En minúsculas, clave |
| display_name | string | |
| last_activity_date | date, nullable | Nulo = nunca ha usado Copilot |

**copilot_usage_user_app** · hechos · usuario x app x foto
Columnas: snapshot_date, report_refresh_date, upn, app, last_activity_date (nullable). `app`: Teams, Outlook, Word, Excel, PowerPoint, Copilot Chat, OneNote, Loop; pueden aparecer otras.

**dim_users** · usuario: upn (clave), display_name, department ("Sin departamento" si vacío), country ("Sin país" si vacío), job_title.

**dim_app** · app: app (clave), orden (int; Teams=1 ... Loop=8, nuevas=99).

**dim_date** · día: date (clave; desde 200 días antes de la primera foto hasta hoy), year, month, month_name ("ene 2026"), week_start (lunes), is_weekday.

`year_month` (year * 100 + month) para ordenar month_name se crea como columna calculada en el modelo (Import).

## Modelo semántico (reglas)
- Modo Import desde el endpoint SQL de LH_Copilot (decisión del 03/11/2026; sustituye a Direct Lake). Hay que refrescar el modelo después de cada carga del notebook.
- Se permiten columnas calculadas (p. ej. `dim_date[year_month]` para ordenar month_name), parámetros what-if, parámetros de campo, tablas DATATABLE desconectadas y grupos de cálculo.
- Se construye y valida con los datos reales del Lakehouse (de momento una sola foto, 28/09/2026). Sin RLS.
- Relaciones uno a varios, filtro en una dirección:
  - dim_date[date] -> snapshot_date de los dos hechos
  - dim_users[upn] -> upn de los dos hechos
  - dim_app[app] -> copilot_usage_user_app[app]
- dim_date marcada como tabla de fechas; dim_app[app] ordenada por `orden`. Columnas técnicas ocultas.
- Formato es-ES: fechas dd/mm/yyyy, euros, porcentajes con 1 decimal.
- Medidas, carpetas y descripciones en español. Toda medida lleva descripción.
- Carpetas de visualización: Base, Adopción, Evolución, Apps, Usuarios, Coste, Textos y formato.

### Definición de activo (no cambiar sin avisar)
Un usuario está activo si ha usado Copilot en cualquier app en los 28 días anteriores a `report_refresh_date` de esa foto. Las medidas se evalúan siempre en la última foto visible en el contexto.

```
Fecha foto = CALCULATE ( MAX ( copilot_usage_user[snapshot_date] ) )

Licencias =
VAR _f = [Fecha foto]
RETURN CALCULATE ( DISTINCTCOUNT ( copilot_usage_user[upn] ), copilot_usage_user[snapshot_date] = _f )

Activos 28d =
VAR _f = [Fecha foto]
RETURN
    CALCULATE (
        COUNTROWS (
            FILTER (
                copilot_usage_user,
                NOT ISBLANK ( copilot_usage_user[last_activity_date] )
                    && DATEDIFF ( copilot_usage_user[last_activity_date], copilot_usage_user[report_refresh_date], DAY ) < 28
            )
        ),
        copilot_usage_user[snapshot_date] = _f
    )

% Adopción = DIVIDE ( [Activos 28d], [Licencias] )
```

Comparaciones "hace 4 semanas": `snapshot_date = [Fecha foto] - 28` con `REMOVEFILTERS ( dim_date )`.

Segmentos por días sin actividad: Activo semanal (< 7), Activo mensual (7-27), En riesgo (28-59), Dormido (>= 60), Nunca usado (sin fecha).

Coste: parámetro "Precio licencia" de 0 a 60 €/usuario/mes, 30 por defecto. Es un supuesto y así hay que presentarlo.

### Validación obligatoria
En la última foto, Licencias y Activos 28d tienen que cuadrar con esta consulta en el endpoint SQL del Lakehouse:
```sql
SELECT snapshot_date, COUNT(*) AS licencias,
       SUM(CASE WHEN DATEDIFF(day, last_activity_date, report_refresh_date) < 28 THEN 1 ELSE 0 END) AS activos_28d
FROM copilot_usage_user GROUP BY snapshot_date ORDER BY snapshot_date DESC
```

## Informe (reglas de diseño)
Paleta Microsoft 365 Live 2026: azul marino #0E1F3D, teal #17A2B8, rojo #E8462D, naranja #F5821F, amarillo #F2B705, verde #4CAF50, azul hielo #CADCFC.

- Semáforo: bueno #4CAF50, neutro #F2B705, malo #E8462D.
- Segoe UI. Fondo de página #F4F7FC; visuales blancos con esquinas redondeadas y sombra suave.
- Lienzo 1280 x 720 con navegación lateral en azul marino.
- Páginas: Resumen ejecutivo, Evolución, Aplicaciones, Licencias y usuarios, Ficha de usuario (obtención de detalles), Análisis libre (parámetros de campo), Tooltip de departamento.
- Privacidad: el informe se proyecta en un evento público. Hay un conmutador "Mostrar nombres / Ocultar nombres" y por defecto los nombres van anonimizados.
- Modelo preparado para Copilot: descripciones en todas las medidas, sinónimos (licencias dormidas, área, aplicación...) e instrucciones de IA.

## Cómo trabajar en este proyecto
- Antes de construir, inspecciona el esquema real del Lakehouse (tipos, nulos, filas, rango de fechas) y avisa de cualquier diferencia con este documento.
- Propón el plan y espera confirmación antes de cambios grandes (esquema de tablas, definición de activo, estructura del modelo).
- No ejecutes el notebook en modo real ni sobrescribas tablas del Lakehouse sin pedirlo.
- Valida las medidas con DAX contra el modelo publicado y revisa visualmente cada página del informe.
- Idioma: español para nombres visibles, descripciones, comentarios y mensajes. Los identificadores de tabla y columna se quedan en snake_case en inglés.
- Respuestas concisas y directas. Sin emojis.
