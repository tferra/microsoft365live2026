# Preguntas de ejemplo para el Data Agent `da_copilot_adoption`

Agente de Fabric en el workspace **M365Live - Adopción Copilot**, sobre el modelo semántico **CopilotAdoption**.

La columna "Respuesta esperada" sale de los datos simulados cargados el 03/11/2026 (última foto 03/11/2026, 84 fotos, 120 licencias). El modo simulado genera las fechas a partir del día de ejecución, así que las cifras cambian cada vez que se relanza el notebook. Antes de la sesión, vuelve a comprobarlas con la consulta SQL de validación de CLAUDE.md.

## 1. Situación actual

| Pregunta | Respuesta esperada |
|---|---|
| ¿Cuál es el porcentaje de adopción de Copilot? | 44,2 % (53 activos de 120 licencias) |
| ¿Cuántas licencias de Copilot tenemos asignadas? | 120 |
| ¿Cuántos usuarios han usado Copilot en los últimos 28 días? | 53 |
| ¿Y en los últimos 7 días? | 42 |
| ¿Cuántas licencias no se han usado nunca? | 58 |
| ¿A qué fecha corresponden los datos? | Foto del 03/11/2026, informe de Graph refrescado el 01/11/2026 |

## 2. Departamentos

| Pregunta | Respuesta esperada |
|---|---|
| ¿Qué departamento tiene más adopción? | IT, 70,0 % |
| ¿Qué área tiene menos adopción? | RRHH, 20,0 % |
| Muéstrame el % de adopción por departamento ordenado de mayor a menor | IT 70 %, Marketing 55 %, Comercial 50 %, Finanzas 35 %, Operaciones 35 %, RRHH 20 % |
| ¿Cuántas licencias sin uso hay en Finanzas? | 13 (20 licencias, 35 % activas) |
| Compara Marketing y Comercial | Marketing 55 % frente a Comercial 50 %, 20 licencias cada uno |

## 3. Aplicaciones

| Pregunta | Respuesta esperada |
|---|---|
| ¿En qué aplicación se usa más Copilot? | Outlook, 51 usuarios activos (42,5 % de las licencias) |
| Ranking de apps por usuarios activos en 28 días | Outlook 51, Teams 49, Copilot Chat 46, Word 43, Excel 38, PowerPoint 30, OneNote 9, Loop 8 |
| ¿Qué apps apenas se usan? | Loop (8) y OneNote (9) |
| ¿Cuántas apps usa de media un usuario activo? | 5,2 |
| ¿Qué porcentaje de licencias usa Copilot en Excel? | 31,7 % |

## 4. Evolución

| Pregunta | Respuesta esperada |
|---|---|
| ¿Cómo ha cambiado la adopción en las últimas 4 semanas? | Sube de 37,5 % a 44,2 %, +6,7 puntos |
| ¿Cuántos usuarios nuevos se han activado respecto a hace un mes? | 11 nuevos activos |
| ¿Qué porcentaje de los activos de hace 4 semanas sigue activo? | Retención 28d del 93,3 % |
| ¿Cuál es el máximo histórico de adopción? | 44,2 %, en la última foto |
| Muéstrame la evolución semanal del % de adopción | Serie con el % de adopción de cada semana (week_start) |

## 5. Licencias y coste

Todas las respuestas de coste usan el precio supuesto de 30 €/usuario/mes. El agente debería decir que es un supuesto.

| Pregunta | Respuesta esperada |
|---|---|
| ¿Cuántas licencias podríamos reasignar? | 67 licencias sin uso en 28 días |
| ¿Cuánto nos ahorraríamos al mes reasignando las licencias dormidas? | 2.010 € al mes |
| ¿Y al año? | 24.120 € |
| ¿Cuánto nos cuesta cada usuario activo? | 67,92 € al mes por usuario activo |
| ¿Cuánto pagamos al mes por Copilot? | 3.600 € (120 x 30 €) |

## 6. Segmentos y usuarios

| Pregunta | Respuesta esperada |
|---|---|
| ¿Cuántos usuarios están en riesgo de abandonar Copilot? | 3 en riesgo (28-59 días sin uso) |
| ¿Cuántos usuarios están dormidos? | 6 (60 días o más sin uso) |
| Reparto de usuarios por segmento | 42 activos semanales, 11 mensuales, 3 en riesgo, 6 dormidos, 58 nunca usado |
| ¿Qué usuarios llevan más tiempo sin usar Copilot? | Códigos U0026 (107 días), U0107 (104), U0011 (94) y U0039 (90), sin nombres |

## 7. Preguntas para enseñar los límites (útiles en directo)

| Pregunta | Comportamiento esperado |
|---|---|
| Dame el nombre y el correo de los usuarios dormidos | No da nombres ni correos. Responde con códigos de usuario (instrucción de privacidad) |
| ¿Qué prompts usa la gente en Copilot? | Explica que estos datos no incluyen el contenido de los prompts |
| ¿Copilot nos ha hecho más productivos? | Explica que mide uso, no productividad |
| ¿Cuánto ahorraríamos si la licencia costara 25 €? | Recalcula con 25 € (Precio licencia) o explica que el precio es un parámetro |

## 8. Guion corto para la demo

1. "¿Cuál es el porcentaje de adopción de Copilot?"
2. "¿Qué área tiene menos adopción y cuántas licencias sin uso tiene?"
3. "¿Cómo ha evolucionado la adopción en el último mes?"
4. "¿En qué aplicaciones se usa más y en cuáles menos?"
5. "¿Cuánto podríamos ahorrar al año reasignando las licencias dormidas?"
6. "Dame el nombre de los usuarios dormidos", para enseñar que respeta la privacidad.
