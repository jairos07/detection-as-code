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

## 2. Mapeo de `logsource` → `if_sid` / `if_group`

El campo `if_sid` (o `if_group`) en Wazuh indica que una regla solo debe evaluarse si el evento ya coincidió con una regla "padre" (normalmente la regla que decodifica el tipo de evento base). Esto evita que la regla se compare contra todo el tráfico de eventos, y es el mapeo que las herramientas comunitarias fallan en hacer correctamente.

**Importante — lección aprendida durante la Fase 1**: esta instalación de Wazuh tiene **dos generaciones de reglas para Sysmon conviviendo en el mismo ruleset**:
- Una cadena "clásica" (`0595-win-sysmon_rules.xml`, reglas `616xx`, usa `<if_sid>`), visible al inspeccionar Management → Rules por nombre/descripción.
- Una cadena moderna y activa en la práctica (`0800-sysmon_id_1.xml`, `0830-sysmon_id_11.xml`, reglas `92xxx`, usa `<if_group>`), que es la que realmente evalúa los eventos reales del agente.

Por tanto, **no basta con identificar una regla candidata por su nombre en el listado de reglas** — es necesario verificarlo generando un evento real y comprobando en el dashboard qué `rule.id` salta de verdad, antes de fijar el `if_sid`/`if_group` de una regla propia. Ver `docs/troubleshooting.md`, problema #9, para el proceso de diagnóstico completo.

| `logsource.category` (Sigma) | `logsource.product` | Fuente del evento | Encadenamiento (Wazuh) | Regla/grupo padre verificado |
|---|---|---|---|---|
| `process_creation` | `windows` | Sysmon Event ID 1 | `<if_group>` | `sysmon_eid1_detections` |

Esta tabla se amplía a medida que se incorporan reglas con otros logsources (ej. `network_connection`, `file_event`, etc.), documentando el encadenamiento real verificado (no solo el candidato inicial) para cada uno.

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
| `\|contains` | Regex simple (motor OSRegex por defecto, sin necesidad de `type`) | `EncodedCommand` |
| `\|contains` (lista, con alternancia) | Regex con alternancia `(opcion1\|opcion2)`, **requiere declarar `type="pcre2"`** en el `<field>` | `<field name="..." type="pcre2">(EncodedCommand\|-enc)</field>` |

### Hallazgo: motor de regex por defecto en Wazuh (OSRegex vs PCRE2)

Al convertir un modificador Sigma `|contains` con una lista de valores alternativos (ej. `CommandLine|contains: [-EncodedCommand, -enc]`), la traducción natural a Wazuh usa una expresión regular con alternancia: `(EncodedCommand|-enc)`.

Sin embargo, el motor de reglas de Wazuh usa por defecto **OSRegex**, su propio motor de expresiones simplificado, no PCRE. OSRegex no soporta la sintaxis de alternancia con paréntesis y barra vertical `(A|B)`, y al guardar una regla que la usa sin más, el editor de reglas devuelve un error genérico y poco descriptivo:

```
Error: Could not upload rule (1113) - XML syntax error
```

Este mensaje no indica en absoluto que el problema sea el tipo de motor de regex, lo cual dificulta el diagnóstico si no se sabe de antemano.

**Solución**: declarar explícitamente `type="pcre2"` en el atributo del `<field>` que use esta sintaxis, para indicarle a Wazuh que use el motor de expresiones regulares PCRE2 (compatible con la alternancia) en vez de OSRegex:

```xml
<field name="win.eventdata.commandLine" type="pcre2">(EncodedCommand|-enc)</field>
```

**Cómo se diagnosticó**: ante el error genérico 1113, se descartó primero un problema de sintaxis XML general validando el archivo con `xmllint --noout` (sin errores), lo que confirmó que el XML era válido y el problema era de validación específica de Wazuh. A partir de ahí, se aisló la causa por bisección: se redujo la regla a su mínima expresión (solo `if_sid` + `description`) y se fueron añadiendo elementos uno a uno (`field` de `image`, luego `field` de `commandLine`, luego `mitre`) hasta reproducir el error, identificando así que el segundo `<field>` —el que usa alternancia regex— era el causante.

**Regla general para este proyecto**: cualquier campo Sigma que use `|contains` con una lista de más de un valor (alternancia) debe convertirse a un `<field>` de Wazuh con `type="pcre2"` explícito.

## 5. Mapeo de tags MITRE ATT&CK

| Sigma (`tags`) | Wazuh (`<mitre><id>`) |
|---|---|
| `attack.t1059.001` | `T1059.001` |

Se elimina el prefijo `attack.` y las letras se convierten a mayúsculas, siguiendo el formato que usa Wazuh internamente en su bloque `<mitre>`.

## 6. Reglas convertidas (registro)

Esta sección se amplía con cada regla nueva, documentando la conversión completa aplicada.

### `proc_creation_win_powershell_encodedcommand.yml`

- **Técnica MITRE**: T1059.001
- **Encadenamiento**: `<if_group>sysmon_eid1_detections</if_group>` (verificado con evento real; el candidato inicial `if_sid: 61603` no era el correcto, ver `troubleshooting.md` #9)
- **Nivel**: `medium` (Sigma) → `10` (Wazuh)
- **Campos mapeados**: `Image` → `win.eventdata.image`, `CommandLine` → `win.eventdata.commandLine` (con `type="pcre2"` por usar alternancia regex, ver hallazgo en sección 4)
- **Regla Wazuh resultante**: ver `rules/wazuh/local_rules.xml`, ID `100010`
- **Estado**: ✅ Validada con detección real end-to-end (evento provocado en el Windows client, alerta confirmada en el dashboard con nivel 10 y descripción correcta)
