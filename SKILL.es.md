name: code-clairvoyant
description: Motor de análisis estático + predicción de errores basado en ML que identifica posibles fallos en tiempo de ejecución, condiciones de carrera, fugas de memoria y vulnerabilidades de seguridad antes de ejecutar el código
version: 2.1.0
author: OpenClaw Team
tags:
  - prediction
  - bugs
  - prevention
  - static-analysis
  - ml
dependencies:
  - python>=3.9
  - nodejs>=16.0.0
  - semgrep>=1.50.0
  - bandit>=1.7.5
  - saas-ai-predictor>=0.8.2
  - typescript>=5.0.0
  - esbuild>=0.19.0
  - flamegraph>=0.3.0
requires_openclaw: \">=2.0.0\"
---

# Code Clairvoyant Skill

## Propósito

Code Clairvoyant analiza codebases para predecir posibles errores y vulnerabilidades antes de que se manifiesten en tiempo de ejecución. Combina matching de patrones, análisis de flujo de datos y modelos ML entrenados con millones de commits de corrección de errores para identificar:

- **Race conditions** en código async/await y multi-hilo
- **Memory leaks** en patrones de asignación de recursos
- **Riesgos de null dereference** por null-safety incompleta
- **Vulnerabilidades de seguridad** (inyección, XSS, path traversal)
- **Antipatrones de rendimiento** (bucles O(n²) en rutas calientes, re-renders innecesarios)
- **Inconsistencia de tipos** que el compilador TypeScript podría pasar por alto
- **Mal uso de API** (rechazos no manejados, manejo de errores faltante)
- **Drift de configuración** (errores dependientes de entorno)

### Casos de Uso Reales

1. **Puerta de Pre-commit**: Ejecutado automáticamente en `git commit` para bloquear código propenso a errores
2. **Integración CI/CD**: Fallar builds cuando las predicciones de alta severidad excedan el umbral
3. **Planificación de Refactor**: Identificar hotspots riesgosos antes de refactoring mayor
4. **Onboarding**: Nuevos desarrolladores aprenden pitfalls específicos del proyecto
5. **Aumentación de Code Review**: Sacar a la luz problemas que revisores podrían omitir
6. **Cuantificación de Deuda Técnica**: Rastrear densidad de predicciones a lo largo del tiempo

## Alcance

### Comandos

- `code-clairvoyant analyze [path]` - Escaneo completo con todos los detectores
- `code-clairvoyant predict --type=<type> [files]` - Predicción dirigida para clase de error específica
- `code-clairvoyant suggest <prediction-id>` - Generar sugerencias de fixes con código diffs
- `code-clairvoyant verify <prediction-id> --test` - Validar predicción con pruebas basadas en propiedad
- `code-clairvoyant review <pr-number> --repo=<url>` - Analizar solo diff de PR
- `code-clairvoyant profile --cache` - Generar perfil de rendimiento del análisis mismo
- `code-clairvoyant baseline` - Establecer línea base de predicción actual (para seguimiento)
- `code-clairvoyant export --format=json|csv|sarif` - Salida para herramientas externas
- `code-clairvoyant train --dataset=<path>` - Reentrenar modelos en dataset de errores personalizado
- `code-clairvoyant revert <prediction-id>` - Rollback de fixes aplicados automáticamente

### Configuración

Crear `~/.openclaw/skills/code-clairvoyant/config.yaml`:

```yaml
thresholds:
  critical: 0.85    # Puntuación de confianza para reportar (0-1)
  warning: 0.65
  info: 0.40
detectors:
  - race_conditions: true
  - memory_leaks: true
  - null_deref: true
  - security: true
  - performance: true
  - type_errors: true
exclude:
  - \"**/node_modules/**\"
  - \"**/dist/**\"
  - \"**/.git/**\"
ml_model: \"default-v3\"  # Opciones: default-v3, custom, ensemble
max_files: 10000        # Límite para entornos con memoria limitada
cache_ttl: 3600        # Segundos para cachear resultados de análisis
```

## Proceso de Trabajo

### 1. Configuración de Entorno

Automático en primera ejecución. Instala:
```bash
# Dependencias instaladas en ~/.openclaw/skills/code-clairvoyant/.venv/
pip install -r requirements.txt
npm ci --only=production
```

Descargar modelos ML (~250MB):
```bash
saas-ai-predictor download-model --name=bug-classifier-v3 --output=./models/
```

### 2. Recopilación de Código

```bash
# Construir lista de archivos respetando .gitignore y excludes de config
code-clairvoyant collect --source=./src --output=manifest.txt
```

Lectura paralela de archivos con límite de memoria:
```bash
# Usa scanner basado en Rust para velocidad
clairvoyant-scanner --manifest=manifest.txt --max-mb=500 --threads=8
```

### 3. Fase de Análisis Estático

**Fase 3a: Matching de Patrones (Reglas Semgrep)**
```bash
# Reglamento OpenClaw personalizado (1200+ reglas)
semgrep --config=./rules/clairvoyant.yaml --json --quiet src/
```

**Fase 3b: Análisis de Flujo de Datos**
```bash
# Construir grafo de control interprocedimental (ICFG)
pyopera analyze --cfg --taint src/ --output=icfg.dot
```

**Fase 3c: Inferencia ML**
```bash
# Extraer features de AST, alimentar modelo
saas-ai-predictor infer --model=models/bug-classifier-v3.pt --features=ast_features.json --batch-size=64
```

### 4. Agregación de Predicciones

Combinar resultados de todas las fases, deduplicar por ubicación, asignar puntuaciones de confianza basadas en:
- Número de detectores que dispararon el problema
- Probabilidad del modelo ML
- Tasa histórica de falsos positivos para ese patrón

### 5. Generación de Sugerencias

Para cada predicción:
```bash
# Generar hasta 3 variantes de fix usando síntesis basada en plantillas + LLM
code-clairvoyant suggest --prediction-id=12345 --strategy=minimal-change
```

### 6. Verificación (Opcional)

Generación de pruebas basadas en propiedad con Hypothesis:
```bash
code-clairvoyant verify --prediction-id=12345 --test-framework=pytest --iterations=1000
```

### 7. Reporte

Formatos de salida:
- **Terminal**: Código de color con file:line:col, severidad, confianza
- **JSON**: Objetos de predicción completos con diffs sugeridos
- **SARIF**: Integración GitHub Code Scanning
- **HTML**: Reporte interactivo con filtro/ordenamiento

Ejemplo de salida terminal:
```
src/utils/auth.js:45:12 [CRITICAL] Posible await faltante en async validateToken()
  Detectado: llamadas a función async sin await
  Confianza: 0.92
  Sugerencia: await validateToken(user) antes de continuar
  Razón ML: \"Patrón: async function devuelve promise pero no se await'a\"
  Ejecutar: code-clairvoyant suggest c49a8f3e
```

## Reglas de Oro

1. **Nunca auto-fix sin `--force` explícito**: Incluso predicciones de alta confianza requieren revisión humana
2. **Siempre mostrar razonamiento**: Incluir qué detector disparó y atribución de features ML
3. **Respetar líneas base del proyecto**: Si predicciones exceden línea base por >20%, investigar causa raíz
4. **Cachear agresivamente**: Reusar resultados de archivos sin cambios (hash SHA256)
5. **Fallar rápido en errores de sintaxis**: Código inválido causa comportamiento indefinido en detectores
6. **Respetar .gitignore**: Nunca analizar archivos generados o dependencias vendor
7. **Proteger secrets**: Nunca loguear o cachear código que contenga API keys/tokens detectados
8. **Limitar rate de sugerencias LLM**: Máximo 5 sugerencias por minuto para evitar throttling de API
9. **Siempre proveer rollback**: Fixes aplicados automáticamente deben ser reversibles en un comando
10. **Documentar falsos positivos**: Añadir `# clairvoyant-ignore: <rule-id>` con justificación

## Ejemplos

### Ejemplo 1: Encontrar race conditions en funciones async

```bash
# Analizar directorio src/ completo
code-clairvoyant analyze src/ --detectors=race_conditions --format=terminal

# Salida:
src/handlers/api.js:78:5 [CRITICAL] Check-then-act no atómico en estado compartido
  Condición de carrera: checking user.sessionActive luego llamando user.refresh()
  Confianza: 0.94 (patrón: CHECK_THEN_ACT_ASYNC)
  Sugerir: Usar lock mutex o atomic compare-and-swap
  code-clairvoyant suggest --id=rc9f2a1b

src/components/Dashboard.tsx:125:15 [WARNING] await faltante en setState dentro de loop
  setState async puede causar race en React 18+ modo concurrente
  Confianza: 0.71 (patrón: ASYNC_SETSTATE_IN_LOOP)
```

### Ejemplo 2: Predecir memory leaks en operaciones intensivas de recursos

```bash
# Objetivo tipos de archivo específicos
code-clairvoyant predict --type=memory_leaks --include=\"**/*.go\" --threshold=0.8

# Salida en JSON para integración IDE:
{
  \"predictions\": [
    {
      \"id\": \"ml_87b3c4d\",
      \"file\": \"server/connection_pool.go\",
      \"line\": 203,
      \"severity\": \"critical\",
      \"confidence\": 0.89,
      \"bug_type\": \"unclosed_http_body\",
      \"description\": \"http.Response.Body puede no cerrarse en camino de error\",
      \"suggestion\": {
        \"strategy\": \"defer_close\",
        \"diff\": \"203: +defer resp.Body.Close()\n 204: +if resp.StatusCode != 200 { return fmt.Errorf(...) }\"
      }
    }
  ]
}
```

### Ejemplo 3: Integración CI/CD

```yaml
# .github/workflows/clairvoyant.yml
jobs:
  bug-prediction:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Code Clairvoyant
        run: |
          code-clairvoyant analyze . --format=sarif --output=results.sarif
          # Subir a GitHub Code Scanning
          github/codeql-action/upload-sarif@v2 --sarif-file=results.sarif
        env:
          CLAIRVOYANT_THRESHOLD: \"0.75\"  # Fallar bajo esta confianza
      - name: Revisar predicciones críticas
        run: |
          CRITICAL=$(code-clairvoyant export --format=json | jq '[.predictions[] | select(.severity==\"critical\")] | length')
          if [ \"$CRITICAL\" -gt 5 ]; then
            echo \"Predicciones críticas exceden umbral: $CRITICAL\"
            exit 1
          fi
```

### Ejemplo 4: Integración bot de PR review

```bash
# Analizar solo archivos cambiados en PR
code-clairvoyant review --repo=https://github.com/owner/repo --pr=456 --token=$GITHUB_TOKEN

# Auto-comentar en PR:
/run-code-clairvoyant --detectors=all --threshold=0.7

# Bot responde:
Code Clairvoyant encontró 3 problemas en este PR:
1. src/lib/db.js:87 - Posible inyección SQL vía parámetro sin escapar
2. src/api/upload.js:45 - Falta validación de tipo de archivo
3. src/utils/payment.js:201 - Rechazo de promesa no manejado
Ver reporte completo: https://clairvoyant.openclaw.io/pr/456
```

### Ejemplo 5: Entrenar modelo custom para patrones específicos del proyecto

```bash
# Exportar errores históricos de Jira/GitHub a formato de entrenamiento
code-clairvoyant export-bugs --since=\"2023-01-01\" --output=training-data/

# Formato: training-data/<bug-id>/buggy_code.ext, training-data/<bug-id>/fixed_code.ext

# Entrenar nuevo modelo
code-clairvoyant train --dataset=training-data/ --epochs=50 --validation-split=0.2

# Salida:
Epoch 1/50: loss=0.342, val_loss=0.415
Epoch 50/50: loss=0.078, val_loss=0.092
Modelo guardado: models/custom-project-v1.pt
Exactitud en validación: 94.2%

# Desplegar modelo custom
code-clairvoyant config set ml_model=custom-project-v1
```

### Ejemplo 6: Seguimiento de línea base para deuda técnica

```bash
# Establecer línea base en rama main
git checkout main
code-clairvoyant baseline --name=v1.0 --output=baseline.json

# Luego, comparar rama feature
git checkout feature/new-api
code-clairvoyant baseline --compare=../main-baseline.json

# Salida:
Comparado contra línea base (v1.0):
  Nuevas predicciones: +12
  Predicciones resueltas: -3
  Cambio neto: +9 (aumento de deuda técnica)
  Densidad de predicción: 3.4 por KLOC (línea base: 2.1)
  
  Nuevos hotspots principales:
    src/api/v2/user_service.go:45:45 [CRITICAL] Falta rate limiter
    src/api/v2/user_service.go:78:12 [WARNING] PII sin encriptar en logs
```

## Comandos de Rollback

### 1. Rollback de un solo fix aplicado automáticamente

```bash
# Listar fixes aplicados
code-clairvoyant history --applied-only

# Salida:
Fixes aplicados (últimos 10):
  ID           Aplicado En             Archivo
  fix_9f2a1b   2026-03-07 14:22:01   src/auth.js
  fix_3c8d4e   2026-03-07 14:20:45   src/api/client.ts

# Revertir fix específico
code-clairvoyant revert fix_9f2a1b
# Restaura src/auth.js desde git HEAD, crea backup en src/auth.js.clairvoyant-backup
```

### 2. Revertir todos los fixes de una sesión

```bash
# Revertir últimos N fixes
code-clairvoyant revert --last=5

# Revertir fixes aplicados antes de timestamp específico
code-clairvoyant revert --before=\"2026-03-07 13:00:00\"
```

### 3. Rollback bulk con verificación

```bash
# Dry run primero
code-clairvoyant revert --pattern=\"*.go\" --dry-run

# Muestra: Revertiría 8 fixes en 5 archivos

# Ejecutar con confirmación
code-clairvoyant revert --pattern=\"src/memory/**\" --confirm
```

### 4. Recuperación basada en Git

```bash
# Si falla comando revert, restaurar desde git
git checkout HEAD -- src/modified/file.js

# O restaurar commit específico
git show <commit-antes-clairvoyant>:src/file.js > src/file.js
```

### 5. Rollback de configuración

```bash
# Si cambios de configuración causan problemas
code-clairvoyant config reset --to-default

# O restaurar backup
cp ~/.openclaw/skills/code-clairvoyant/backups/config-20260307.yaml ~/.openclaw/skills/code-clairvoyant/config.yaml
```

## Variables de Entorno

| Variable | Requerida | Default | Descripción |
|----------|-----------|---------|-------------|
| `CLAIRVOYANT_THRESHOLD` | No | 0.65 | Puntuación de confianza mínima (0-1) para reportar predicciones |
| `CLAIRVOYANT_CACHE_DIR` | No | `~/.cache/clairvoyant` | Sobrescribir ubicación de cache |
| `CLAIRVOYANT_MODEL_PATH` | No | `./models/` | Ruta a modelos ML personalizados |
| `CLAIRVOYANT_MAX_WORKERS` | No | `auto` | Hilos worker para análisis (1-N) |
| `CLAIRVOYANT_LLM_API_KEY` | No | - | API key para generación de sugerencias (OpenAI/Anthropic) |
| `CLAIRVOYANT_DISABLE_ML` | No | `false` | Poner `true` para usar solo detectores basados en reglas |

## Dependencias

### Requisitos Python (`requirements.txt`)
```
semgrep==1.50.0
bandit==1.7.5
saas-ai-predictor @ git+https://github.com/openclaw/saas-ai-predictor@v0.8.2
networkx==3.2
astroid==3.0.1
hypothesis==6.82.0
pytest==7.4.0
flask==3.0.0
sqlalchemy==2.0.23
```

### Requisitos Node.js (`package.json`)
```json
{
  \"dependencies\": {
    \"typescript\": \"^5.0.0\",
    \"esbuild\": \"^0.19.0\",
    \"@typescript-eslint/parser\": \"^6.0.0\",
    \"eslint\": \"^8.0.0\",
    \"flamegraph\": \"^0.3.0\"
  }
}
```

### Dependencias de Sistema
- `graphviz` (para visualización CFG)
- `ripgrep` (escaneo rápido de archivos)
- `git` (para historia/comparación línea base)

## Pasos de Verificación

### Verificación post-instalación

```bash
# Verificar todos los componentes
code-clairvoyant check

# Salida esperada:
[OK] Python 3.11.5 encontrado
[OK] Node.js 18.17.0 encontrado
[OK] Semgrep 1.50.0 instalado
[OK] Modelo ML descargado (247MB)
[OK] Directorio cache escribible
[OK] Configuración válida
Todos los sistemas listos ✓
```

### Verificar con bug conocido

```bash
# Crear archivo de prueba con bug intencional
cat > test_race.js << 'EOF'
let counter = 0;
async function increment() {
  const temp = counter;
  await new Promise(r => setTimeout(r, 1));
  counter = temp + 1; // Race condition
}
EOF

# Ejecutar análisis
code-clairvoyant analyze test_race.js --format=json

# Debería detectar:
# predicción \"race_condition\" en línea 4 con confianza >0.8
```

### Verificación CI

```bash
# Simular ejecución CI
code-clairvoyant analyze src/ --format=sarif --output=ci-results.sarif
cat ci-results.sarif | grep -q '\"level\":\"error\"' && echo \"Problemas críticos encontrados\" || echo \"Limpio\"
```

## Solución de Problemas

### Problema: \"MemoryError durante análisis de codebase grande\"

**Solución**: Reducir count de workers o habilitar modo incremental
```bash
code-clairvoyant analyze src/ --max-workers=2 --incremental
# O aumentar cache TTL para reusar resultados
export CLAIRVOYANT_CACHE_TTL=7200
```

### Problema: \"Modelo ML no encontrado / descarga falló\"

**Solución**: Descargar manualmente o usar modo solo-reglas
```bash
# Saltar ML (menos preciso pero funciona)
code-clairvoyant analyze src/ --disable-ml

# O descargar manualmente:
saas-ai-predictor download-model --url=https://cdn.openclaw.ai/models/v3/bug-classifier.pt
```

### Problema: \"Tasa de falsos positivos >30%\"

**Solución**: Ajustar umbral o entrenar modelo custom
```bash
# Subir umbral para reducir ruido
code-clairvoyant analyze src/ --threshold=0.8

# O recolectar falsos positivos y reentrenar:
code-clairvoyant mark-fp --prediction-id=abc123 --reason=\"falso positivo: lock está realmente held\"
code-clairvoyant train --dataset=$(code-clairvoyant export-fp-dataset) --fine-tune
```

### Problema: \"Demasiado lento (>10 minutos)\"

**Solución**: Habilitar caching, limitar detectores, excluir archivos
```bash
# Primera ejecución será lenta, siguientes ejecuciones con cache
code-clairvoyant analyze src/ --cache-ttl=86400

# Deshabilitar detectores costosos
code-clairvoyant analyze src/ --detectors=race_conditions,security --skip=performance

# Excluir archivos de prueba
echo \"**/*.test.*\" >> ~/.openclaw/skills/code-clairvoyant/.ignore
```

### Problema: \"Detector crashea en sintaxis inválida\"

**Solución**: Pre-validar todos los archivos, arreglar errores de sintaxis primero
```bash
# Encontrar errores de sintaxis
find src/ -name \"*.js\" -exec node --check {} \; 2>&1 | grep -v \"SyntaxError\"

# O dejar que clairvoyant omita archivos inválidos:
code-clairvoyant analyze src/ --skip-invalid-syntax --strict=false
```

### Problema: \"Operaciones de historia git muy lentas\"

**Solución**: Deshabilitar comparación línea base o limitar rango de historia
```bash
code-clairvoyant baseline --days=7  # Solo mirar última semana
# O saltar git completamente:
code-clairvoyant analyze src/ --no-git-context
```
```