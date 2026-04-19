# Revisión funcional del repositorio

## ¿Qué hace este proyecto?
Este repositorio implementa un prototipo de **Dependency Analysis Platform (DAP)** para SSDLC. Su objetivo es escanear dependencias Python, normalizar hallazgos de seguridad y generar un reporte con priorización de riesgo.

Flujo principal:
1. Lee un `requirements.txt` (o un JSON de entrada).
2. Ejecuta `pip-audit` (o procesa JSON de `pip-audit` / `semgrep`).
3. Normaliza vulnerabilidades a un esquema común.
4. Calcula puntaje de riesgo ponderado y nivel de riesgo.
5. Persiste resultados en `db_scan_results.json`.

## Componentes clave

### `src/scanner.py`
- `run_pipaudit_scan(...)`: ejecuta `pip-audit` con salida JSON.
- `load_json_report(...)`: carga y normaliza reportes ya existentes.
- `enrich_vulnerabilities(...)`: añade metadatos de estado y timestamp.
- `build_report(...)`: construye el resultado final con resumen por severidad, score y estado.
- `main()`: CLI para orquestar el escaneo (`--requirements`, `--input-json`, `--input-type`).

### `src/parser.py`
- `normalize_tool_report(...)`: adapta salida de `pipaudit` y `semgrep` al formato estándar.
- `infer_severity(...)`: inferencia básica de severidad cuando no viene explícita.
- `calculate_custom_risk_score(...)`: modelo simple de scoring (CRITICAL=10, HIGH=7, MODERATE=4, LOW=1).

### Pruebas
- `tests/test_scanner.py` valida que el reporte marque estado `FAILED` con vulnerabilidades y score correcto.
- `tests/test_parser.py` valida normalización y cálculo de score.

## Hallazgos importantes (revisión técnica)
1. **Bug menor de duplicación de clave** en `build_report`: la clave `status` se define dos veces en el diccionario; Python conserva la última, por lo que no rompe ejecución, pero es deuda técnica.
2. **Manejo de errores mejorable** al invocar `pip-audit`: no se valida `returncode` ni se procesa `stderr` para dar mensajes accionables.
3. **Modelo de severidad heurístico** en `infer_severity`: útil para demo, pero puede sobre/sous-estimar sin CVSS real.
4. **Cobertura de tests básica**: cubre casos núcleo, pero faltan pruebas de errores de subprocess, entradas malformadas y rama `semgrep` más completa.


## ¿Cómo afectan estos 4 hallazgos al código (impacto real)

### 1) Duplicación de la clave `status` en `build_report`
**Dónde ocurre:** `src/scanner.py` dentro del diccionario de retorno de `build_report`.

**Impacto técnico:**
- Python sobrescribe la primera clave con la segunda, así que hoy el valor final es correcto por casualidad (no por diseño).
- Aumenta el riesgo de errores silenciosos: si en el futuro ambas líneas difieren, una de ellas se ignorará sin aviso.
- Complica el mantenimiento y la lectura del código, porque no está claro cuál `status` “manda”.

**Impacto de negocio/operación:**
- Puede generar reportes inconsistentes en cambios futuros y provocar decisiones equivocadas de triage en CI/CD.

### 2) Manejo de errores insuficiente al ejecutar `pip-audit`
**Dónde ocurre:** `run_pipaudit_scan` en `src/scanner.py` (no valida `returncode` ni trata `stderr`/JSON inválido).

**Impacto técnico:**
- Si `pip-audit` falla (comando ausente, error de red, formato inesperado), el parseo de salida puede fallar o devolver datos vacíos sin contexto útil.
- El flujo no diferencia con claridad entre “no hay vulnerabilidades” y “falló el escaneo”.

**Impacto de negocio/operación:**
- Riesgo de **falsos negativos operativos**: pipeline aparentemente limpio cuando en realidad el escaneo no se ejecutó bien.
- Dificulta soporte y debugging por falta de mensajes accionables.

### 3) Heurística de severidad simplificada (`infer_severity`)
**Dónde ocurre:** `src/parser.py` (asigna HIGH a CVE/PYSEC y MODERATE al resto).

**Impacto técnico:**
- No refleja fielmente el riesgo real si la fuente no trae severidad explícita.
- Puede sesgar el `weighted_risk_score` y por tanto el `risk_level` agregado.

**Impacto de negocio/operación:**
- Priorización incorrecta: issues críticos podrían quedar subestimados o issues moderados sobreestimados.
- Afecta la confianza del equipo en el score para decidir bloqueos o excepciones.

### 4) Cobertura de pruebas limitada
**Dónde ocurre:** `tests/` (hay pruebas núcleo, pero faltan escenarios de error y ramas menos ejercitadas).

**Impacto técnico:**
- Mayor probabilidad de regresiones al tocar subprocess, parseo de errores o integración de formatos alternativos.
- Menor detección temprana de bugs en caminos “no felices”.

**Impacto de negocio/operación:**
- Se incrementa el costo de mantenimiento y el riesgo de incidentes en pipeline al crecer el proyecto.
- Reduce la velocidad de cambios seguros porque cada refactor exige validación manual adicional.

## Valor del repositorio
- Es una base clara para demostrar integración SSDLC (scanner + scoring + salida auditable + tests).
- Está preparado para evolucionar hacia controles más robustos (gates por umbral, persistencia real, tracking de estados).

## Sugerencias de evolución
- Eliminar duplicación de `status` en `build_report`.
- Agregar manejo explícito de errores de `pip-audit` (timeout, comando faltante, JSON inválido).
- Integrar severidad CVSS cuando exista en la fuente.
- Incrementar tests de integración del flujo CLI y casos negativos.
