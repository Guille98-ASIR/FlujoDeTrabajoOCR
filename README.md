# Protocolo de Trabajo: Sistema de Digitalización y OCR Masivo

## Resumen Ejecutivo

Protocolo estándar para proyectos de OCR de corpus documentales mediante API de Anthropic. Desarrollado en colaboración con la Facultad de Filología de la Universidad de Sevilla, considerando las particularidades de la investigación filológica.

**Institución de referencia:** Universidad de Sevilla - Facultad de Filología  
**Aplicación:** Instituciones académicas y centros de investigación

---

## Arquitectura del Sistema


### 1. Sistema de Registro y Catalogación (Google Sheets)

**Función:** Base de datos centralizada para gestión del corpus documental.

**Campos obligatorios:**
- `id_documento`: Identificador único (formato: AAAA-TIPO-NNNN)
- `titulo_documento`: Denominación completa
- `ruta_drive`: ID de Google Drive
- `estado_procesamiento`: pendiente/en_proceso/completado/validado/error
- `fecha_ingreso` y `fecha_procesamiento`
- `responsable`: Investigador asignado

**Campos específicos para filología:**
- `lengua_principal` y `variante_linguistica`
- `periodo_historico` y `genero_textual`
- `estado_conservacion` y `observaciones_paleograficas`
- `nivel_dificultad` (1-5)
- `requiere_revision_experta` (booleano)
- `url_resultado_ocr`

**Hojas complementarias:**
- Estadísticas del proyecto
- Taxonomía de documentos
- Registro de validaciones
- Glosario de términos especializados


### 2. Repositorio Documental (Google Drive)

**Estructura jerárquica recomendada:**

```
📁 Proyecto_OCR_Filologia_[Institución]
├── 📁 00_Documentación
├── 📁 01_Corpus_Original
│   ├── 📁 Por_Periodo (Siglo XVI, XVII, XVIII, etc.)
│   └── 📁 En_Validación
├── 📁 02_Corpus_Procesado
│   ├── 📁 Textos_OCR_Raw
│   └── 📁 Textos_Validados
├── 📁 03_Material_Auxiliar (glosarios, muestras caligráficas)
├── 📁 04_Control_Calidad (revisiones, errores, casos especiales)
└── 📁 05_Logs_Sistema
```

**Consideraciones:**
- Nomenclatura que incluya datación y tipología
- Metadatos en propiedades de archivo
- Control de versiones para documentos corregidos
- Permisos diferenciados por rol
- Protocolo de backup periódico


### 3. Infraestructura Computacional (Ubuntu Server)

**Especificaciones mínimas:**
- Ubuntu 22.04 LTS o superior
- Python 3.10+
- 16GB RAM (recomendado)
- 50GB almacenamiento libre
- Conexión estable

**Dependencias principales:**
```
anthropic>=0.25.0
google-auth>=2.22.0
google-api-python-client>=2.95.0
gspread>=5.10.0
pandas>=2.0.0
Pillow>=10.0.0
pdf2image>=1.16.3
python-dotenv>=1.0.0
```

**Consideraciones para filología:**
Claude tiene capacidad superior en:
- Reconocimiento de variantes paleográficas
- Interpretación de abreviaturas históricas
- Manejo de ortografía no estandarizada
- Preservación de diacríticos

Es fundamental configurar prompts que incluyan contexto lingüístico, convenciones de transcripción y glosarios especializados.


## Procedimiento de Implementación

### Fase I: Preparación del Entorno

```bash
# Crear y activar entorno virtual
mkdir ~/proyectos_ocr/filologia_[institucion]
cd ~/proyectos_ocr/filologia_[institucion]
python3 -m venv .venv
source .venv/bin/activate

# Instalar dependencias
pip install -r requirements.txt
```

### Fase II: Configuración

**Archivo `.env` con parámetros esenciales:**
```bash
# APIs
ANTHROPIC_API_KEY=sk-ant-api03-...
ANTHROPIC_MODEL=claude-sonnet-4-20250514

# Google Cloud
GOOGLE_APPLICATION_CREDENTIALS=./config/service_account.json
SPREADSHEET_ID=1abc...xyz
DRIVE_FOLDER_ID_ORIGINALS=1xyz...abc

# Procesamiento
MAX_CONCURRENT_JOBS=3
BATCH_SIZE=20
TIMEOUT_PER_DOCUMENT=300

# Filología
CORPUS_LANGUAGES=es,la,ca
PRESERVE_ORIGINAL_ORTHOGRAPHY=true
INCLUDE_PALEOGRAPHIC_NOTES=true
```

**Credenciales Google Cloud:**
1. Crear proyecto en console.cloud.google.com
2. Habilitar APIs: Drive y Sheets
3. Crear cuenta de servicio → Descargar JSON
4. Compartir Drive y Sheets con email de cuenta de servicio

**Validar configuración:**
```bash
python scripts/test_setup.py
```


## Arquitectura del Software

**Estructura modular:**
```
proyecto_ocr_filologia/
├── .venv/                    # Entorno virtual
├── .env                      # Configuración
├── requirements.txt          # Dependencias
├── config/
│   ├── settings.py          # Configuración central
│   ├── prompts/             # Templates de prompts
│   └── service_account.json # Credenciales GCP
├── src/
│   ├── core/                # Lógica de OCR
│   ├── integrations/        # APIs (Anthropic, Google)
│   ├── models/              # Modelos de datos
│   └── utils/               # Utilidades
├── scripts/
│   ├── process_batch.py     # Script principal
│   ├── validate_catalog.py  # Validación
│   └── generate_statistics.py
├── logs/                    # Logs de ejecución
└── docs/                    # Documentación
```

---

## Protocolo de Trabajo Operativo

### Fase 1: Preparación del Corpus (Institución colaboradora)

**Responsabilidad:** Equipo investigador de la institución

**1.1. Digitalización y organización**

El equipo investigador debe:

- Digitalizar documentos originales con resolución mínima de 300 DPI
- Aplicar nomenclatura sistemática a archivos (ej: `1550_epistola_fernandezOropesa_001.pdf`)
- Cargar documentos en la carpeta correspondiente de Google Drive
- Organizarlos según criterios filológicos establecidos (cronología, tipología, autor, etc.)

**1.2. Catalogación en hoja de cálculo**

Para cada documento, registrar:

- Metadatos bibliográficos completos
- Información paleográfica relevante
- Particularidades lingüísticas conocidas
- Dificultades previstas para OCR
- Prioridad de procesamiento (alta/media/baja)

**1.3. Revisión previa**

Antes de iniciar el procesamiento:

- Verificar que todos los documentos son accesibles
- Confirmar calidad de las digitalizaciones
- Asegurar que los metadatos son completos y consistentes
- Establecer criterios de transcripción específicos del proyecto

**Coordinación:** Reunión inicial con el equipo técnico para alinear expectativas y resolver dudas metodológicas.

### Fase 2: Configuración del Proyecto

**Responsabilidad:** Equipo técnico

**2.1. Inicialización del entorno**

```bash
cd ocr-filologia-base

# Renombrar proyecto
mv ocr-filologia-base ocr-filologia-[institucion]
cd ocr-filologia-[institucion]

# Inicializar entorno virtual
python3 -m venv .venv
source .venv/bin/activate

# Instalar dependencias
pip install -r requirements.txt
```

**2.2. Configuración específica del proyecto**

```bash
# Copiar plantilla de configuración
cp config/.env.example .env

# Editar configuración (usar editor preferido)
nano .env
```

Completar con:
- Credenciales de API de Anthropic
- IDs de Google Drive y Sheets específicos del proyecto
- Parámetros ajustados a las características del corpus
- Configuración de idiomas y características paleográficas

**2.3. Personalización de prompts**

Adaptar los templates en `config/prompts/` según:

- Lenguas presentes en el corpus
- Convenciones ortográficas de la época
- Sistema de abreviaturas utilizado
- Criterios de transcripción establecidos por los investigadores

**Ejemplo de prompt especializado:**

```
Eres un experto paleógrafo especializado en documentos españoles del siglo XVII.

CONTEXTO DEL DOCUMENTO:
- Lengua: Castellano del Siglo de Oro
- Tipo: Correspondencia administrativa
- Características: Uso extensivo de abreviaturas estándar de la época

INSTRUCCIONES DE TRANSCRIPCIÓN:
1. Transcribe respetando la ortografía original (no modernices)
2. Desarrolla abreviaturas siguiendo estas convenciones:
   - q̃ → que
   - V.M. → Vuestra Merced
   - dho → dicho
   [agregar lista completa específica del proyecto]
3. Marca pasajes ilegibles como [ilegible]
4. Indica inserciones o correcciones del autor como ^[texto]
5. Respeta puntuación original, incluida ausencia de puntuación
6. Mantén mayúsculas/minúsculas del documento original

FORMATO DE SALIDA:
- Texto transcrito en párrafos
- Anotaciones paleográficas entre [corchetes]
- Metadatos estructurados al final
```

**2.4. Validación del sistema**

```bash
# Ejecutar prueba con muestra pequeña
python scripts/process_batch.py --test-mode --sample-size 5

# Verificar resultados
python scripts/validate_catalog.py --check-recent
```

### Fase 3: Procesamiento Masivo del Corpus

**Estrategia de procesamiento por tiers:**
- **Tier 1:** Documentos alta prioridad → supervisión inmediata
- **Tier 2:** Corpus principal → procesamiento automatizado, revisión por muestreo
- **Tier 3:** Material complementario → procesamiento masivo, validación diferida

**Ejecución:**
```bash
# Procesamiento básico
python scripts/process_batch.py --filter estado=pendiente --workers 3

# Con validación automática
python scripts/process_batch.py --validate-output --quality-threshold 0.85

# Documentos específicos
python scripts/process_batch.py --document-ids 1550-EPIST-001,1550-EPIST-002
```

**Casos especiales:** Documentos con confianza <70%, múltiples manos, deterioro significativo se marcan automáticamente para revisión experta.


### Fase 4: Control de Calidad

**Validación técnica:**
```bash
python scripts/validate_catalog.py --comprehensive
python scripts/generate_statistics.py --quality-metrics
```

**Validación filológica:**
Muestra estratificada (5-10% por estrato) evaluando:
- Fidelidad ortográfica (≥95%)
- Desarrollo de abreviaturas (≥90%)
- Preservación de estructura
- Calidad de anotaciones paleográficas

**Corrección iterativa:** Análisis de errores → Ajuste de prompts → Reprocesamiento → Validación

### Fase 5: Entrega

```bash
# Organizar resultados
python scripts/organize_final_output.py --structure chronological

# Estadísticas finales
python scripts/generate_statistics.py --output docs/informe_final.pdf
```

**Documentación:**
- Informe técnico (metodología, estadísticas, problemas)
- Guía de uso del corpus (estructura, convenciones, limitaciones)
- Sesión de transferencia de conocimiento con equipo investigador

---

## Marco de Seguridad

**Clasificación de datos:**
- Nivel 1-4: Público → Interno → Confidencial → Restringido

**Gestión de credenciales:**
- ✅ NUNCA commitear `.env`, `service_account.json` a Git
- ✅ Usar variables de entorno
- ✅ Rotar API keys cada 3-6 meses
- ✅ Principio de mínimo privilegio
- ✅ Mantener audit trail

**Protección de datos:**
- En tránsito: TLS 1.3
- En reposo: Permisos restrictivos, cifrado de backups
- En API: Políticas de retención, cumplimiento GDPR

**Auditoría:** Logging comprehensivo de accesos, invocaciones API, modificaciones, exportaciones y errores.

---

## Métricas y Monitorización

**KPIs técnicos:** Throughput, disponibilidad, tasa de error, latencia
**KPIs de calidad:** Precisión OCR, desarrollo abreviaturas, coherencia formato
**KPIs de proyecto:** Adherencia cronograma, cobertura corpus, eficiencia costos

**Dashboard en Google Sheets:**
Fórmulas básicas para tracking automático:
- Total documentos: `=CONTAR(A:A)-1`
- Completados: `=CONTAR.SI(G:G,"completado")`  
- Progreso: `=Completados/Total*100`
- Tiempo promedio: `=PROMEDIO(I:I-H:H)`

**Visualizaciones:**
```bash
python scripts/generate_statistics.py --visualizations
# Genera gráficos de progreso, calidad, cobertura y tiempos
```

---


## Resolución de Problemas

### Errores comunes

**Error de autenticación Google Cloud:**
```bash
# Verificar configuración
echo $GOOGLE_APPLICATION_CREDENTIALS
ls -la config/service_account.json
chmod 600 config/service_account.json
```
Solución: Verificar que carpetas Drive y Sheets están compartidas con email de cuenta de servicio.

**Rate limiting API Anthropic:**
```bash
# Reducir paralelización
python scripts/process_batch.py --workers 1 --delay 2.0
```
Implementar backoff exponencial con librería `tenacity`.

**Calidad OCR insuficiente:**

Causas y soluciones:
- **Baja resolución:** Re-digitalizar a ≥300 DPI
- **Contraste pobre:** Preprocesar imagen (binarización, nitidez)
- **Prompt genérico:** Crear prompts ultra-específicos con:
  - Muestras de caligrafía del escriba
  - Lista exhaustiva de abreviaturas
  - Contexto histórico y temático
  - Ejemplos de desarrollos correctos

**Procesamiento iterativo:** Primera pasada → Análisis errores → Ajuste prompt → Reprocesamiento

### Casos especiales filológicos

**Documentos multilingües:** Identificar lengua por sección, marcar cambios `[LATÍN]...[/LATÍN]`

**Anotaciones marginales:** Distinguir texto principal de marginalia con formato estructurado

**Elementos no textuales:** Describir sellos, firmas, rúbricas con metadatos detallados

---


## Recursos

**Documentación técnica:**
- Anthropic: https://docs.anthropic.com
- Google Cloud: https://developers.google.com/drive, /sheets

**Plantillas disponibles:**
- Spreadsheet para catálogo filológico
- Estructura de carpetas Drive
- Templates de prompts especializados
- Scripts de utilidad

---

## Guía de Adaptación para Nuevas Instituciones

### Proceso de onboarding (4-6 semanas)

**Semanas 1-2: Evaluación**
- Reunión inicial (características corpus, objetivos, cronograma)
- Análisis de muestra documental
- Estimación de esfuerzo

**Semanas 3-4: Setup técnico**
- Configuración GCP y credenciales
- Desarrollo de prompts especializados
- Procesamiento piloto (20-30 docs)
- Evaluación y ajustes

**Semanas 5+: Procesamiento masivo**
- Ejecución por lotes
- Validación continua
- Cierre y documentación

### Checklist de inicio

**Preliminar:**
- [ ] Corpus digitalizado (≥300 DPI)
- [ ] Metadatos documentados
- [ ] Criterios de transcripción definidos
- [ ] Presupuesto y cronograma aprobados

**Técnico:**
- [ ] Proyecto GCP + APIs habilitadas
- [ ] Cuenta de servicio configurada
- [ ] Estructura Drive creada
- [ ] Spreadsheet con permisos
- [ ] API key Anthropic activa
- [ ] Entorno Python + dependencias
- [ ] Variables de entorno configuradas

**Piloto:**
- [ ] Prompts iniciales redactados
- [ ] 20-30 documentos procesados
- [ ] Validación por equipo investigador
- [ ] Calidad aceptable confirmada

### Personalización por corpus

**Epistolarios:** Énfasis en fórmulas cortesía, datación, posdata
**Protocolos notariales:** Fórmulas jurídicas, otorgantes, cláusulas
**Documentación administrativa:** Sellos, registros, firmas
**Literatura manuscrita:** Versificación, variantes, correcciones de autor
**Documentación eclesiástica:** Abreviaturas latinas, títulos, datación litúrgica

### Consideraciones para filología

Los departamentos de filología requieren:
1. **Rigor académico:** Transcripciones citables, trazabilidad
2. **Respeto material:** No modernizar, preservar irregularidades
3. **Flexibilidad metodológica:** Adaptación a criterios diversos
4. **Interoperabilidad:** Formatos estándar (TEI, plain text)
5. **Expertise humana:** OCR asiste, no reemplaza al filólogo

---

## Contacto y Versión

**Coordinación académica:**
Universidad de Sevilla - Facultad de Filología

**Repositorio:** https://github.com/[org]/ocr-filologia

| Versión | Fecha | Cambios |
|---------|-------|---------|
| 1.0 | 2026-02-16 | Versión inicial - Proyecto Universidad de Sevilla |

---

© 2026 [Organización]  
Desarrollado en colaboración con Facultad de Filología, Universidad de Sevilla

**Última actualización:** 16 febrero 2026
