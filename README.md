# ¿Está funcionando Copilot en tu empresa? Mide la adopción con Microsoft Fabric

Demo de la sesión de **Microsoft 365 Live 2026** (online, 4 de noviembre de 2026).
Speaker: Toni Ferrà, Data Tech Lead en Bravent.

Un notebook de Fabric descarga cada día de Microsoft Graph los informes de uso de Microsoft 365 Copilot y guarda una foto diaria en un Lakehouse. Con ese histórico se construyen un modelo semántico, un informe de Power BI de adopción y un Data Agent.

> La API de Graph solo da la foto del periodo. El valor está en guardar la película: el histórico diario.

## Contenido

| Ruta | Qué es |
|---|---|
| `nb_ingesta_copilot_usage.ipynb` | Notebook de ingesta Graph -> Lakehouse (modo `simulado` o `real`) |
| `pbip/CopilotAdoption.pbip` | Proyecto Power BI (PBIP) |
| `pbip/CopilotAdoption.SemanticModel/` | Modelo semántico en TMDL (Import desde el endpoint SQL del Lakehouse) |
| `pbip/CopilotAdoption.Report/` | Informe en PBIR, con tema e iconos en `StaticResources` |
| `preguntas-data-agent.md` | Preguntas de ejemplo para el Data Agent |
| `M365Live2026 - Adopcion de Copilot con Fabric.pptx` | Presentación de la sesión |
| `CLAUDE.md` | Documentación técnica detallada: tablas, reglas del modelo y del informe |

## Arquitectura

```
Microsoft Graph (informes de uso de Copilot + usuarios)
        |  notebook diario (7:00, hora de Madrid)
        v
Lakehouse LH_Copilot (tablas Delta: foto diaria)
        |  endpoint SQL
        v
Modelo semántico CopilotAdoption (Import)
        |
        +--> Informe Power BI "Adopción de Copilot"
        +--> Data Agent da_copilot_adoption
```

### Tablas Delta

- `copilot_usage_user`: hechos, un registro por usuario y foto (fecha de última actividad).
- `copilot_usage_user_app`: hechos, un registro por usuario, app y foto.
- `dim_users`, `dim_app`, `dim_date`: dimensiones.

### Definición de activo

Un usuario está **activo** si ha usado Copilot en cualquier app en los 28 días anteriores a la fecha de refresco del informe de Graph. `% Adopción = Activos 28d / Licencias`, siempre evaluado en la última foto visible.

### Informe

Páginas: Resumen ejecutivo, Evolución, Aplicaciones, Licencias y usuarios, Ficha de usuario (obtención de detalles), Análisis libre (parámetros de campo) y Tooltip de departamento. Por defecto los nombres de usuario salen anonimizados; un conmutador permite mostrarlos.

El coste usa el parámetro "Precio licencia" (30 €/usuario/mes por defecto). Es un supuesto, no un precio oficial.

## Puesta en marcha

### Prueba rápida sin tenant (modo simulado)

1. Crea en Fabric un workspace con capacidad F y un Lakehouse `LH_Copilot` (sin esquemas).
2. Importa el notebook y adjunta el Lakehouse como predeterminado.
3. Ejecútalo con `MODO = "simulado"`: genera 84 fotos diarias de 120 usuarios ficticios con la misma estructura que devuelve Graph.

### Datos reales

1. Registra una aplicación en Entra ID con los permisos de aplicación de Graph `Reports.Read.All` y `User.Read.All`, y concede consentimiento de administrador.
2. Guarda en Key Vault los secretos `copilot-tenant-id`, `copilot-client-id` y `copilot-client-secret`.
3. En el notebook, cambia `KEY_VAULT_URL` (`https://<tu-keyvault>.vault.azure.net/`) y pon `MODO = "real"`.
4. En el centro de administración de Microsoft 365, desactiva los nombres ocultos en los informes. Si no, los UPN llegan como hashes y no cruzan con `dim_users`.
5. Programa el notebook a diario. Cada ejecución sustituye la foto del día, así que relanzarlo no duplica datos.

### Modelo e informe

1. En `pbip/CopilotAdoption.SemanticModel/definition/expressions.tmdl`, sustituye `<tu-endpoint-sql>` por el host del endpoint SQL de tu Lakehouse.
2. Abre `pbip/CopilotAdoption.pbip` con Power BI Desktop, refresca y publica en el workspace.
3. Refresca el modelo después de cada carga del notebook.

### Validación

En la última foto, `Licencias` y `Activos 28d` deben cuadrar con esta consulta en el endpoint SQL:

```sql
SELECT snapshot_date, COUNT(*) AS licencias,
       SUM(CASE WHEN DATEDIFF(day, last_activity_date, report_refresh_date) < 28 THEN 1 ELSE 0 END) AS activos_28d
FROM copilot_usage_user GROUP BY snapshot_date ORDER BY snapshot_date DESC
```
