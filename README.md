# Detection-as-Code

**Pipeline de CI/CD para reglas de detección con cobertura MITRE ATT&CK medida automáticamente**

## El problema

En la mayoría de los SOC, las reglas de detección se escriben, se despliegan y se "espera que funcionen". No existe un proceso sistemático que demuestre, de forma repetible, que una regla detecta realmente la técnica de ataque para la que fue diseñada. Cuando cambia el entorno, se actualiza el SIEM o pasa el tiempo, nadie vuelve a comprobar si esa regla sigue funcionando. El resultado es una falsa sensación de cobertura: se cree estar protegido frente a técnicas que en realidad ya no se detectan.

## La solución

Este proyecto trata cada regla de detección como código: versionada en Git, validada automáticamente contra un ataque real simulado, y desplegada solo si supera esa validación — igual que un test unitario en desarrollo de software, pero aplicado a la ingeniería de detección.

## Cómo funciona

1. Las reglas de detección se escriben en **Sigma**, el estándar abierto de la industria, y se versionan en un repositorio Git.
2. Al hacer commit de una regla nueva o modificada, se dispara un **pipeline de CI/CD en GitLab CE** que:
   - convierte la regla Sigma al formato de Wazuh y la despliega vía API,
   - lanza el test de ataque correspondiente contra un laboratorio de Active Directory, usando **Atomic Red Team** (técnicas individuales) o **Caldera** (cadenas de ataque completas),
   - consulta la API de Wazuh para comprobar si se generó la alerta esperada,
   - **falla el pipeline si la regla no detecta el ataque**, bloqueando su paso a producción.
3. Con cada validación exitosa, se actualiza automáticamente una **capa de MITRE ATT&CK Navigator**, mostrando qué técnicas están cubiertas, con qué confianza y cuándo se validaron por última vez.
4. Las reglas se publican en abierto, con el objetivo de contribuir a repositorios comunitarios de Sigma.

## Stack tecnológico

| Herramienta | Función en el proyecto |
|---|---|
| Sigma | Formato de reglas de detección, agnóstico de SIEM |
| Wazuh | Motor de detección / SIEM donde se despliegan y ejecutan las reglas |
| Atomic Red Team | Emulación de técnicas individuales de MITRE ATT&CK |
| Caldera | Emulación de cadenas de ataque completas |
| GitLab CE | Control de versiones y pipeline de CI/CD |
| MITRE ATT&CK Navigator | Visualización de cobertura de detección |

## Qué demuestra este proyecto

- Comprensión de ingeniería de detección más allá de "copiar reglas de internet"
- Capacidad de automatizar procesos de validación de seguridad (no solo desplegar, sino demostrar)
- Conocimiento práctico de MITRE ATT&CK como marco de referencia, no solo como diagrama
- Familiaridad con CI/CD aplicado a un contexto no habitual (detección, no desarrollo de software)
- Disciplina de documentación e ingeniería reproducible

## Estado del proyecto

- [x] Laboratorio de Active Directory operativo
- [x] Wazuh desplegado y recibiendo telemetría
- [ ] Reglas Sigma iniciales
- [ ] Pipeline de CI/CD en GitLab CE
- [ ] Integración con Atomic Red Team
- [ ] Generación automática de capa MITRE ATT&CK Navigator
- [ ] Integración con Caldera
- [ ] Publicación / contribución comunitaria

## Licencia

Este proyecto se publica bajo licencia MIT.