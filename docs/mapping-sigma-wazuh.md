# Mapeo manual Sigma → Wazuh

## Por qué existe este documento

No existe un backend oficial de pySigma para Wazuh en el directorio de plugins de SigmaHQ. Las alternativas comunitarias existentes (p. ej. `pysigma-backend-wazuh`) no mapean correctamente el campo `if_sid`, lo que provoca que las reglas convertidas se evalúen contra todos los eventos que procesa el manager, en lugar de limitarse al logsource correcto — es incorrecto funcionalmente y costoso en rendimiento.

Ante este hueco, este proyecto define y documenta su propio proceso de conversión manual: cada regla Sigma se traduce a mano a una regla nativa de Wazuh (XML), siguiendo las convenciones descritas en este documento. Esto garantiza control total sobre la conversión y sirve como referencia reproducible para cualquier regla futura del repositorio.

## 1. Equivalencia de niveles de severidad

Sigma define 5 niveles de severidad (`level`), mientras que Wazuh usa una escala numérica de 0 a 16. No existe un estándar oficial de conversión entre ambos sistemas, así que este proyecto define la siguiente tabla, usada de forma consistente en todas las reglas:

| Nivel Sigma | Nivel Wazuh | Justificación |
|---|---|---|
| `informational` | 3 | Solo aporta contexto, sin indicio de riesgo |
| `low` | 6 | Anómalo pero muy probablemente benigno |
| `medium` | 10 | Sospechoso, merece revisión de un analista |
| `high` | 12 | Alta probabilidad de actividad maliciosa |
| `critical` | 15 | Evidencia casi confirmada de compromiso |

El nivel `16` se reserva deliberadamente para alertas correlacionadas de múltiples fuentes o reglas compuestas, no para reglas individuales de detección directa.

## 2. Mapeo de `logsource` → `if_sid`

El campo `if_sid` en Wazuh indica que una regla solo debe evaluarse si el evento ya coincidió con una regla "padre" (normalmente la regla que decodifica el tipo de evento base). Esto evita que la regla se compare contra todo el tráfico de eventos, y es el mapeo que las herramientas comunitarias fallan en hacer correctamente.

Este proyecto identifica el `if_sid` correspondiente inspeccionando el ruleset real cargado en la instancia de Wazuh (Management → Rules), en lugar de asumir valores estándar de la documentación genérica, ya que estos pueden variar según la versión del ruleset instalado.

| `logsource.category` (Sigma) | `logsource.product` | Fuente del evento | `if_sid` (Wazuh) | Regla base |
|---|---|---|---|---|
| `process_creation` | `windows` | Sysmon Event ID 1 | `61603` | Sysmon - Event 1: Process creation |

Esta tabla se amplía a medida que se incorporan reglas con otros logsources (ej. `network_connection`, `file_event`, etc.), documentando el `if_sid` real encontrado en el ruleset para cada uno.

## 3. Mapeo de nombres de campo

Los campos de una regla Sigma usan nombres genéricos (taxonomía Sigma). Wazuh, para eventos de Sysmon, expone estos mismos datos bajo el prefijo `win.eventdata.*`. La siguiente tabla registra las equivalencias usadas en este proyecto, ampliándose con cada nueva regla:

| Campo Sigma | Campo Wazuh | Notas |
|---|---|---|
| `Image` | `win.eventdata.image` | Ruta completa del ejecutable |
| `CommandLine` | `win.eventdata.commandLine` | Línea de comandos completa del proceso |

## 4. Mapeo de modificadores Sigma → sintaxis Wazuh

| Modificador Sigma | Equivalente en Wazuh | Ejemplo |
|---|---|---|
| `\|endswith` | Regex anclada al final (`$`) dentro de `<field>` | `powershell\.exe$` |
| `\|contains` | Regex sin anclar, o `type="pcre2"` si se requiere coincidencia parcial | `EncodedCommand` |
| `\|contains` (lista) | Alternancia regex (`opcion1\|opcion2`) o múltiples `<field>` combinados con `<if_sid>` compartido | `(EncodedCommand\|-enc)` |

## 5. Mapeo de tags MITRE ATT&CK

| Sigma (`tags`) | Wazuh (`<mitre><id>`) |
|---|---|
| `attack.t1059.001` | `T1059.001` |

Se elimina el prefijo `attack.` y las letras se convierten a mayúsculas, siguiendo el formato que usa Wazuh internamente en su bloque `<mitre>`.

## 6. Reglas convertidas (registro)

Esta sección se amplía con cada regla nueva, documentando la conversión completa aplicada.

### `proc_creation_win_powershell_encodedcommand.yml`

- **Técnica MITRE**: T1059.001
- **`if_sid`**: 61603
- **Nivel**: `medium` (Sigma) → `10` (Wazuh)
- **Campos mapeados**: `Image` → `win.eventdata.image`, `CommandLine` → `win.eventdata.commandLine`
- **Regla Wazuh resultante**: ver `rules/wazuh/local_rules.xml`, ID `100010`
