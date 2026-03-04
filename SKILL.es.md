name: rapid-executor
description: Ejecutor impulsado por IA para tareas de análisis con ejecución paralela y lógica de reintentos inteligente
version: 1.2.0
author: OpenClaw Team
tags:
  - analysis
  - ai
  - automation
  - execution
  - performance
dependencies:
  - python>=3.9
  - jq>=1.6
  - parallel>=20161222
  - git
  - bash>=5.0
required_tools:
  - rapid-exec
  - rapid-analyze
  - rapid-cache
os:
  - linux
  - darwin
---

# Rapid Executor

Motor de ejecución impulsado por IA para tareas de análisis con paralelización inteligente, procesamiento por lotes dinámico y estrategias de reintento adaptativas.

## Propósito

Ejecutar canalizaciones de análisis complejas en múltiples objetivos con utilización óptima de recursos. Diseñado para:
- Análisis paralelo de repositorios (escaneos simultáneos RPGCLAW y FlickClaw)
- Procesamiento por lotes de archivos de registro desde múltiples servidores VPS
- Escaneos concurrentes de vulnerabilidades de dependencias con manejo automático de límites de velocidad
- Ejecución distribuida de pruebas con agregación de resultados
- Benchmarking de rendimiento a través de variantes de entorno

## Alcance

### Comandos Principales

```
# Ejecutar tarea de análisis con procesamiento por lotes impulsado por IA
rapid-exec --task <task_name> --targets <targets_file> [--ai-optimize] [--max-parallel N]

# Analizar repositorio con detección inteligente
rapid-analyze --repo <path|url> [--depth N] [--include-tests] [--output-format json|tariff]

# Gestión de caché para ejecuciones repetidas
rapid-cache --policy <lru|ttl|size> --max-size <bytes>

# Ejecución distribuida vía SSH
rapid-exec --ssh-hosts <hosts_file> --command "<cmd>" [--env-file <env>]

# Reintento inteligente con retroceso exponencial
rapid-exec --task <task> --retry-attempts N --retry-delay <seconds>
```

### Archivos de Configuración

- `~/.config/rapid/executor.conf` - Configuración principal
- `.rapidignore` - Patrones de exclusión (similar a .gitignore)
- `rapid-tasks.yaml` - Definiciones de tareas con parámetros sugeridos por IA

## Proceso de Trabajo Detallado

### 1. Fase de Descubrimiento de Objetivos
- Escanear fuentes de entrada (archivo, stdin, --targets-file)
- Deduplicar objetivos usando hashing consciente de contenido
- Clasificar objetivos por tipo (repositorio, servidor, log, suite-de-pruebas)
- Estimar requisitos de recursos por objetivo

### 2. Fase de Optimización IA (si --ai-optimize habilitado)
- Analizar datos históricos de ejecución de rapid-cache
- Predecir tiempo de ejecución por objetivo basado en:
  - Tamaño del repositorio (git rev-list --count HEAD)
  - Conteo de archivos (find . -type f | wc -l)
  - Complejidad de dependencias (parsing de package.json/go.mod/pyproject.toml)
- Generar tamaños de lote óptimos para maximizar paralelización sin agotar recursos
- Crear plan de ejecución con orden prioritario

### 3. Motor de Ejecución
```bash
# Flujo de ejecución real:
INPUT_TARGETS=$(cat $TARGETS_FILE | grep -v '^#' | grep -v '^$')
TOTAL=${#INPUT_TARGETS[@]}

# Cálculo de lote optimizado por IA
BATCH_SIZE=$(rapid-calc-optimal-batch --targets $TOTAL --memory $AVAIL_RAM --cpu $CPU_CORES)

# Ejecución paralela con seguimiento de progreso
parallel --jobs $MAX_PARALLEL --joblog rapid.log --progress \
  "rapid-run-single {} --context $EXECUTION_ID" ::: ${INPUT_TARGETS[@]}

# Agregación en tiempo real
tail -f rapid.log | rapid-aggregate --format json > results/$TASK_NAME.json
```

### 4. Procesamiento de Resultados
- Validar códigos de salida (no-cero = fallo)
- Capturar stdout/stderr a caché con hash del objetivo
- Generar resumen ejecutivo con insights de IA:
  - Objetivos más lentos identificados
  - Patrones de fallo detectados
  - Cuellos de botella de recursos destacados
- Almacenar en formato estructurado (JSON con metadatos)

### 5. Reporte
- Salida de consola con spinner/barra de progreso
- Directorio `results/` con salidas por objetivo
- `summary.json` con métricas agregadas
- Webhook opcional a sistema CI/CD

## Reglas de Oro

1. **Nunca deshabilitar --ai-optimize para conjuntos grandes (>50 objetivos)** - El procesamiento por lotes manual causa agotamiento de recursos
2. **Siempre usar autenticación SSH con clave** - Los prompts de contraseña rompen la ejecución paralela
3. **La caché es autoritativa** - Nunca eliminar `~/.cache/rapid/` sin `rapid-cache --evict-all`
4. **El archivo de objetivos debe estar separado por nuevas líneas, una por línea** - Espacios/comentarios causan fallos silenciosos
5. **La propagación de códigos de salida es obligatoria** - Siempre verificar `$?` después de rapid-exec
6. **Los límites de memoria deben establecerse** - El análisis paralelo sin límites causa OOM
7. **Los archivos de registro deben rotarse** - rapid.log crece indefinidamente; usar `logrotate`
8. **Las definiciones de tareas deben estar bajo control de versiones** - Almacenar `rapid-tasks.yaml` en el repositorio
9. **Formato de archivo de hosts SSH: `user@host:port`** - Sin puerto predeterminado a 22
10. **Nunca ejecutar rapid-exec como root** - Crea problemas de permisos en caché

## Ejemplos

### Ejemplo 1: Analizar todos los microservicios RPGCLAW
```bash
# Descubrir servicios
find services/ -name package.json -exec dirname {} \; > targets.txt

# Ejecutar con optimización IA
rapid-exec \
  --task dependency-scan \
  --targets targets.txt \
  --ai-optimize \
  --max-parallel 8 \
  --retry-attempts 3 \
  --retry-delay 5

# Salida:
# ✓Ejecutados 42/42 objetivos en 3m17s (promedio: 4.7s)
# ⚠ 2 objetivos excedieron 10s: user-service, payment-service
# ✗ Fallaron: legacy-auth (puerto 3000 sin respuesta)
# Tasa de aciertos de caché: 78%
```

### Ejemplo 2: Análisis de registros paralelo en flota VPS
```bash
# hosts.txt:
# admin@vps-rpgclaw-1.example.com:22
# admin@vps-rpgclaw-2.example.com:22
# admin@vps-flickclaw-1.example.com:3010

rapid-exec \
  --ssh-hosts hosts.txt \
  --command "journalctl --since '2 hours ago' | grep -i error" \
  --env-file .env.ssh \
  --output-format jsonlines \
  --max-parallel 4

# Resultado guardado en: results/log-analysis-20240115.jsonl
# Cada línea: {host: "vps-rpgclaw-1", errors: 127, samples: [...]}
```

### Ejemplo 3: Auto-reintento con retroceso para pruebas de red intermitentes
```bash
rapid-analyze \
  --repo ./tests/e2e \
  --include-tests \
  --retry-attempts 5 \
  --retry-delay 2 \
  --retry-backoff multiplicative \
  --max-parallel 2

# Usa IA para identificar pruebas con fallos intermitentes
# Reejecuta automáticamente solo las pruebas fallidas
# Reporte final muestra estadísticas de reintento por prueba
```

### Ejemplo 4: Ejecución repetida consciente de caché
```bash
# Primera ejecución (caché fría)
time rapid-exec --task lint --targets repos.txt
# real    5m23s

# Segunda ejecución (caché caliente, sin cambios de código)
time rapid-exec --task lint --targets repos.txt
# real    1m48s  (66% más rápido vía aciertos de caché)

# Limpiar solo entradas antiguas (>7 días)
rapid-cache --policy ttl --max-age 604800
```

### Ejemplo 5: Tarea personalizada con parámetros sugeridos por IA
```yaml
# rapid-tasks.yaml
tasks:
  security-scan:
    command: "npm audit --json"
    ai_suggestion:
      max_parallel: "target_count < 20 ? target_count : 20"
      retry_attempts: 2
      timeout: 300  # seconds
      memory_per_target_mb: 256
```

rapid-exec --task security-scan --targets services.txt --ai-optimize
# IA calcula: 42 servicios → tamaño de lote 20, 3 lotes paralelos
```

## Comandos de Rollback

### Aborto Inmediato (propagación SIGINT/SIGTERM)
```bash
# Enviar SIGTERM a todos los procesos hijos
pkill -P $$ rapid-run-single
# O usar integrado:
rapid-exec --abort --execution-id <id_from_log>
```

### Limpiar Resultados Parciales
```bash
# Eliminar artefactos de ejecución incompletos
rm -rf results/$TASK_NAME-*
rm -f rapid.log rapid.partial

# Limpiar caché para tarea específica
rapid-cache --evict-task $TASK_NAME --targets-file <file>
```

### Invalidación Total de Caché
```bash
# Conservador: eliminar solo entradas más antiguas que X
rapid-cache --policy ttl --max-age 3600

# Nuclear: borrar toda la caché
rapid-cache --evict-all
# La reconstrucción de caché ocurrirá en la siguiente ejecución
```

### Restaurar Estado Anterior desde Backup
```bash
# Snapshots diarios de caché en ~/.cache/rapid/backups/
# Listar backups disponibles
ls ~/.cache/rapid/backups/

# Restaurar backup específico
rapid-restore-cache --backup ~/.cache/rapid/backups/20240115_0200.tar.gz

# Después de restaurar, volver a ejecutar solo tareas fallidas
rapid-exec --task $TASK_NAME --targets failed.txt --resume
```

### Rollback de Hosts SSH (si el comando modificó estado remoto)
```bash
# Backup pre-ejecución (antes de ejecutar rapid-exec)
for host in $(cat hosts.txt); do
  ssh $host "pg_dump -Fc mydb > /backups/mydb-$(date +%s).dump" &
done
wait

# Después de detección de fallo:
# rapid-exec detecta salida no-cero, activa automáticamente:
rapid-rollback-ssh --hosts hosts.txt --backup-pattern "/backups/mydb-*.dump"
```

### Restauración de Cuota de Recursos
```bash
# Si rapid-exec modificó límites del sistema:
# ej., ulimit -n 65536 o sysctl net.core.somaxconn

# Restaurar desde línea base almacenada
rapid-restore-limits --baseline /etc/security/limits.d/rapid-baseline.conf

# O manual:
sysctl -p /etc/sysctl.conf
ulimit -n 1024  # Predeterminado
```

### Rollback de Pipeline CI/CD
```bash
# Si rapid-exec activó despliegues descendientes
# Verificar metadatos de ejecución:
cat results/$TASK_NAME/metadata.json | jq .deployment_id

# Usar ID de despliegue para rollback vía sistema de despliegue
deployment-cli rollback $(jq -r .deployment_id results/*/metadata.json)
```

### Verificación post-rollback
```bash
# 1. Confirmar que no quedan hijos rapid-exec
pkill -f "rapid-run-single" && echo "Limpiado"

# 2. Verificar consistencia de caché
rapid-cache --verify

# 3. Verificar espacio en disco recuperado
du -sh ~/.cache/rapid/

# 4. Probar con objetivo único
rapid-exec --task $TASK_NAME --targets <test_target> --max-parallel 1
```

## Solución de Problemas

### Alto uso de memoria a pesar de --max-parallel
- Verificar: `ps aux | grep rapid-run-single | wc -l` (debería igualar max-parallel)
- Causa probable: los objetivos tienen crecimiento de memoria sin límites (node --max-old-space-size no establecido)
- Solución: Establecer `RAPID_MEMORY_LIMIT_MB` en env o flag `--memory-limit`

### Fallos de conexión SSH
- Verificar formato hosts.txt: `user@host:port` (puerto opcional)
- Probar individual: `ssh -i ~/.ssh/id_rsa admin@host -p 22 echo ok`
- Usar `--ssh-options "-o ConnectTimeout=10"` para fallar rápido

### Fallos de caché a pesar de no haber cambios de código
- Rapid-exec usa hash de contenido del directorio objetivo (git HEAD + mtimes de archivos)
- Si es repo git, verificar `git status` - cambios no comprometidos invalidan caché
- Forzar reuso de caché: `rapid-exec --ignore-changes` (peligroso, usar solo para clones idénticos)

### Sugerencias de IA subóptimas
- Verificar datos de entrenamiento: `ls ~/.cache/rapid/ai-metrics/`
- Reconstruir: `rapid-ai-train --historical-logs rapid.log.*.gz`
- Deshabilitar IA: `--ai-optimize false` (cae en valores conservadores predeterminados)

### Fallos silenciosos en ejecución paralela
- Siempre verificar `rapid.log` para códigos de salida individuales
- Usar `--strict-mode` para fallar lote completo si algún objetivo falla
- Habilitar verbose: `RAPID_LOG_LEVEL=debug rapid-exec ...`
```