# Entendiendo este repositorio (guía rápida)

## ¿Qué es?
Este proyecto implementa un **prototipo de análisis de dependencias** para SSDLC. Su objetivo es detectar vulnerabilidades, normalizar resultados de herramientas de seguridad y calcular una prioridad de riesgo accionable.

## Flujo principal
1. **Entrada**: `requirements.txt` (o un JSON importado).
2. **Escaneo**: `src/scanner.py` ejecuta `pip-audit` o consume un JSON existente.
3. **Normalización**: `src/parser.py` convierte la salida de herramientas a un formato común.
4. **Riesgo**: se calcula `weighted_risk_score` y `risk_level`.
5. **Salida**: se genera `db_scan_results.json`.

## Archivos clave
- `src/scanner.py`
  - Orquesta el escaneo.
  - Maneja errores de ejecución/JSON inválido.
  - Enriquece findings con estado y timestamps.
  - Construye el reporte final.
- `src/parser.py`
  - Detecta severidad por CVSS o heurísticas.
  - Filtra vulnerabilidades por rangos de versión afectados.
  - Deduplica hallazgos y conserva el más completo.
  - Normaliza entradas de `pip-audit` y `semgrep`.
- `tests/`
  - Verifica score de riesgo, severidades, deduplicación y filtros por versión.

## Modelo de riesgo usado
- `CRITICAL` = 10
- `HIGH` = 7
- `MODERATE` = 4
- `LOW` = 1

El estado general se marca como:
- `FAILED` si hay vulnerabilidades.
- `PASSED` si no hay hallazgos.

## Cómo ejecutarlo rápido
```bash
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt
pip install pip-audit
python -m src.scanner
cat db_scan_results.json
```

## Qué mejorar después (siguientes pasos sugeridos)
- Persistir resultados en base de datos en vez de archivo JSON único.
- Añadir baseline y diff entre escaneos.
- Agregar soporte para más ecosistemas (`npm`, `maven`, etc.).
- Exponer métricas y tendencias de riesgo por sprint/release.
