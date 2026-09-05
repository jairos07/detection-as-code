k# Troubleshooting

Registro de problemas reales encontrados durante el desarrollo del proyecto, cómo se diagnosticaron y cómo se resolvieron. El objetivo de este documento es doble: servir de referencia rápida si el problema vuelve a aparecer, y dejar evidencia del proceso de diagnóstico seguido (no solo la solución final).

---

## Fase 1 — Reglas Sigma manuales

### 1. `fish shell` no reconoce `venv/bin/activate`

**Síntoma**:
```
source venv/bin/activate
venv/bin/activate (line 48): Unsupported use of '='. In fish, please use 'set _OLD_VIRTUAL_PATH "$PATH"'.
```

**Causa**: el script `venv/bin/activate` que genera Python está escrito en sintaxis de shells POSIX (bash/zsh: `VAR="valor"`), pero fish usa una sintaxis distinta para asignar variables (`set VAR "valor"`), así que no puede interpretarlo directamente.

**Solución**: usar el script de activación específico para fish que el propio `venv` genera junto al genérico:
```fish
source venv/bin/activate.fish
```

---

### 2. `sigma --version` no existe como opción

**Síntoma**:
```
sigma --version
Error: No such option '--version'.
```

**Causa**: la versión instalada de `sigma-cli` no expone `--version` como flag global del comando.

**Solución**: usar el comando estándar de pip para consultar la versión de cualquier paquete instalado:
```fish
pip show sigma-cli
```

---

### 3. No existe backend oficial de Wazuh en pySigma/sigma-cli

**Síntoma**: `sigma plugin list` no muestra ningún backend llamado `wazuh` entre los disponibles.

**Causa**: no existe un backend oficial mantenido por SigmaHQ para Wazuh (confirmado por hilos abiertos sin resolver en el repositorio de pySigma). Existen alternativas comunitarias no oficiales (ej. `pysigma-backend-wazuh`), pero con limitaciones documentadas: no mapean correctamente el campo `if_sid`, lo que provoca que las reglas convertidas se evalúen contra todos los eventos del manager en lugar de limitarse al logsource correcto.

**Solución adoptada**: en vez de depender de un backend automático no oficial y con fallos conocidos, este proyecto define y documenta su propio proceso de conversión manual Sigma → Wazuh, registrado en `docs/mapping-sigma-wazuh.md`. Esto incluye identificar manualmente el `if_sid` correcto inspeccionando el ruleset real de la instancia (Management → Rules en el dashboard), en lugar de asumir valores genéricos de la documentación.

---

### 4. El `id` de una regla Sigma no es el `id` de una regla Wazuh

**Síntoma**: al escribir la primera regla Wazuh, se usó por error el UUID de la regla Sigma (`id: f2c11f9d-...`) como `id` de la regla `<rule>` en Wazuh.

**Causa**: confusión entre dos sistemas de identificación distintos. Sigma usa UUIDs (identificadores largos, universales) en su campo `id`. Wazuh usa números enteros simples, con rangos reservados (0-99999 reglas del sistema; 100000+ para reglas locales propias).

**Solución**: usar un ID numérico propio dentro del rango reservado a reglas locales (`100000+`) para cada regla `<rule>` de Wazuh, dejando el UUID únicamente en el archivo `.yml` de Sigma.

---

### 5. El panel "Ruleset Test" del dashboard no reproduce la cadena de decoders de Sysmon

**Síntoma**: al simular un evento JSON manual con los campos `win.eventdata.image` y `win.eventdata.commandLine` ya completados, el test muestra "Phase 2: Completed decoding" pero nunca llega a una fase de coincidencia de reglas — la regla con `if_sid` apuntando a la regla base de Sysmon (`61603`) nunca se dispara.

**Causa**: el evento JSON simulado a mano se procesó con el decoder genérico `json`, no con la cadena real de decodificación que Wazuh aplica a eventos que llegan desde el canal de Sysmon de Windows a través del agente. Como la regla depende de `<if_sid>61603</if_sid>` (que exige que el evento haya coincidido antes con la regla real de "Sysmon Event 1: Process creation"), y esa regla nunca se evalúa con un JSON simulado a mano, la regla propia tampoco se llega a evaluar.

**Solución / aprendizaje**: el "Ruleset Test" de la GUI es útil para validar sintaxis y ver cómo se decodifican campos sueltos, pero **no es fiable para validar reglas con `if_sid` encadenado a decoders específicos de un canal de eventos** (como Sysmon). La única validación fiable en estos casos es generar el evento real desde el endpoint correspondiente (en este caso, ejecutar el comando PowerShell real en el Windows client/server del lab) y comprobar la alerta resultante en el dashboard.

---

### 6. Error al guardar `local_rules.xml`: "File could not be updated, it already exists" (código 1905)

**Síntoma**:
```
Error: Could not upload rule (1905) - File could not be updated, it already exists
```

**Causa**: se intentó crear un archivo nuevo llamado `local_rules.xml` usando la opción "Add new rules file", cuando ese archivo ya existía por defecto en la instalación de Wazuh (con una regla de ejemplo sobre autenticación SSH fallida).

**Solución**: en vez de crear un archivo nuevo, localizar el `local_rules.xml` ya existente desde "Manage rules files", abrirlo en modo edición, y añadir el bloque `<group>` propio a continuación del contenido ya existente (sin necesidad de eliminar el ejemplo por defecto, ya que cada `<group>` es un bloque independiente).

---

### 7. Error al guardar: "XML syntax error" (código 1113) por alternancia regex sin declarar el tipo de motor

**Síntoma**:
```
Error: Could not upload rule (1113) - XML syntax error
```
El error persistía incluso después de validar el archivo completo con `xmllint --noout` sin encontrar ningún problema de sintaxis XML.

**Causa**: el campo `<field name="win.eventdata.commandLine">(EncodedCommand|-enc)</field>` usa una expresión regular con alternancia (paréntesis + barra vertical), que es sintaxis de motores tipo PCRE/ERE. El motor de reglas de Wazuh usa por defecto **OSRegex**, su propio motor de expresiones simplificado, que no soporta esta sintaxis de alternancia sin indicarlo explícitamente. El mensaje de error que devuelve la API de Wazuh es genérico ("XML syntax error") y no indica en absoluto que el problema real sea el motor de regex, lo que dificulta mucho el diagnóstico a simple vista.

**Cómo se diagnosticó**: 
1. Se descartó primero un problema de sintaxis XML general, validando el archivo completo con `xmllint --noout` (sin errores) — esto confirmó que el XML era válido y el problema era de validación específica del lado de Wazuh, no de sintaxis XML pura.
2. Se aisló la causa por bisección: se redujo la regla a su mínima expresión (solo `if_sid` + `description`, sin ningún `<field>` ni `<mitre>`) y guardó sin problema.
3. Se fueron añadiendo elementos de uno en uno (primer `<field>` de `image` → guardó bien; segundo `<field>` de `commandLine` con alternancia → falló), acotando así el problema a ese campo concreto.

**Solución**: declarar explícitamente `type="pcre2"` en el atributo del `<field>` que necesite usar alternancia regex, para indicarle a Wazuh que use el motor PCRE2 en lugar de OSRegex:
```xml
<field name="win.eventdata.commandLine" type="pcre2">(EncodedCommand|-enc)</field>
```

**Regla general para este proyecto**: cualquier campo Sigma que use `|contains` con una lista de más de un valor (alternancia) debe convertirse a un `<field>` de Wazuh con `type="pcre2"` explícito. Documentado también en `docs/mapping-sigma-wazuh.md`, sección 4.

---

### 8. Guardar cambios en `local_rules.xml` no aplica los cambios inmediatamente

**Síntoma**: tras guardar correctamente el archivo, el dashboard muestra el aviso *"Changes will not take effect until a restart is performed"*.

**Causa**: Wazuh separa el guardado del archivo de reglas (escritura en disco) de su recarga en el motor de reglas activo del manager, por razones de estabilidad (evita recargar el ruleset en caliente ante cada guardado).

**Solución**: pulsar el botón **"Restart"** que aparece junto al aviso, para que el manager recargue el ruleset actualizado y la nueva regla quede activa de verdad.

---

### 9. La regla no salta a pesar de que Sysmon genera el evento y el agente lo envía correctamente: dos generaciones de reglas Sysmon conviviendo en el mismo ruleset

**Síntoma**: tras confirmar que Sysmon generaba el evento (`Get-WinEvent` mostraba el "Process Create" real) y que el agente lo transmitía sin errores al manager (log del agente sin errores, "Analyzing event log: Microsoft-Windows-Sysmon/Operational"), la regla propia (`100010`, con `<if_sid>61603</if_sid>`) seguía sin generar ninguna alerta al buscar `rule.id: 100010` en el dashboard. Ni siquiera la propia regla base `61603` aparecía en las búsquedas.

**Causa raíz**: la instalación de Wazuh tenía **dos generaciones distintas de reglas para Sysmon Event ID 1 conviviendo en el mismo ruleset**, sin que esto fuera evidente a simple vista:

- Una cadena "clásica", en `0595-win-sysmon_rules.xml`, con reglas en el rango `616xx` (ej. `61603`, identificada al principio de la fase inspeccionando Management → Rules), que usa `<if_sid>` para encadenar reglas hijas.
- Una cadena más moderna y específica, en archivos como `0800-sysmon_id_1.xml` y `0830-sysmon_id_11.xml`, con reglas en el rango `92xxx` (ej. `92057`, `92213`), que usa `<if_group>` en lugar de `<if_sid>`, agrupando bajo grupos como `sysmon_eid1_detections`.

El evento real generado por el agente se decodificaba y evaluaba a través de la cadena **moderna** (`92xxx` / `if_group`), no de la clásica (`616xx` / `if_sid`) que se había identificado inicialmente como referencia. Al construir la regla propia con `<if_sid>61603</if_sid>`, esta nunca llegaba a evaluarse, porque el evento no pasaba por esa regla padre en la práctica — aunque la regla `61603` existiera y apareciera correctamente listada en el buscador de reglas del dashboard.

**Cómo se diagnosticó**:
1. Se confirmó extremo a extremo que el evento sí llegaba y se procesaba: `Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational"` mostró el evento real "Process Create" en la máquina Windows.
2. Se revisó el log del agente (`ossec.log`) y se confirmó conexión activa con el manager y lectura del canal de Sysmon, sin errores.
3. Se buscó en el dashboard cualquier alerta reciente del agente sin filtrar por regla propia, y aparecieron alertas de reglas del sistema (`92057` "Powershell.exe spawned a powershell process which executed a base64 encoded command", `92213` "Executable file dropped in folder commonly used by malware") coincidiendo justo con el momento de la prueba — confirmando que el evento sí se decodificaba y evaluaba, solo que contra otras reglas.
4. Se inspeccionó el detalle de esas reglas (`92057`, `92213`) desde Management → Rules, comparando su `Groups`/`If_group` (`sysmon_eid1_detections`, archivo `0800-sysmon_id_1.xml`) con el `if_sid` usado en la regla propia (`61603`, de un archivo distinto y una generación de reglas distinta).

**Solución**: cambiar la regla propia para encadenarse a la cadena de reglas real y activa en la instalación, usando `<if_group>` en lugar de `<if_sid>`:

```xml
<rule id="100010" level="10">
  <if_group>sysmon_eid1_detections</if_group>
  <field name="win.eventdata.image">powershell\.exe$</field>
  <field name="win.eventdata.commandLine" type="pcre2">(EncodedCommand|-enc)</field>
  <description>Ejecucion de PowerShell con parametro -EncodedCommand (posible ofuscacion de comando)</description>
  <mitre>
    <id>T1059.001</id>
  </mitre>
</rule>
```

**Regla general para este proyecto**: antes de fijar el `if_sid`/`if_group` de una regla nueva basándose en el listado de Management → Rules, conviene **verificar con una prueba real qué regla del sistema salta de verdad** con el tipo de evento objetivo (provocando el evento y observando qué `rule.id` aparece en el dashboard), en lugar de asumir que la primera regla "candidata" encontrada por nombre/descripción es la que efectivamente está activa en la cadena de decodificación de la instalación. Este mismo método de verificación se aplicará a las siguientes reglas del proyecto antes de darlas por buenas.
