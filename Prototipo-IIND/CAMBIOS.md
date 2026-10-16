# Registro de cambios

Todo cambio al repositorio se anota aquí, el más reciente arriba. Es obligatorio (ver `CLAUDE.md`, sección 4).

## Formato de cada entrada

```
## AAAA-MM-DD · <autor> · rama <rama> · commit <hash corto>
- Qué cambió:
- Por qué:
- Archivos:
- Cómo se verificó: <salida real de pytest / backtest / CLI>
- Consultado con SOKA: sí (fecha y qué aprobó) / no aplica (cambio pequeño de la sección 2)
- Cifras nuevas y su origen: <de qué pestaña o cálculo sale cada una> / ninguna
- Modelo de Claude usado: <id del modelo>
```

---

## 2026-10-16 · Julián Ordoñez con Claude · rama feat/julian-modelos · reglas v1.2
- Qué cambió: (1) seis modelos entrenados con la base: propensión por ramo (regresión logística L1 con selección por estabilidad), segmentos (k-medias), respuesta al push (beta-binomial con Bayes empírico), demanda por ramo (tendencia lineal), momento de compra (χ²) y tamaño del piloto (z de dos proporciones). (2) Reglas v1.2: el valor esperado se multiplica por la afinidad del modelo (probabilidad / tasa base, acotada a [0,5; 2]) solo en los ramos que pasan la prueba de permutación (SOAT, Vida y Desempleo); el modelo de push puede mandar la oferta a banner con un umbral de 20 %. (3) Pestaña «Modelos» en el panel, columna de afinidad en la traza, línea «Clientes como tú» en la oferta y segmento en Tu año. (4) Imágenes tomadas de capturas de la app real: foto del banner de seguros, ilustraciones de crédito, tarjeta y CDT, y logo de Seguros Mundial.
- Por qué: la fuerza de la señal era una hipótesis; los modelos la reemplazan por evidencia donde la base la tiene y dejan explícito dónde no.
- Archivos: `scripts/train_models.py`, `lulo_app/ml/` (`__init__.py`, `vector.py`, `models.py`, `modelos.json`), `lulo_app/core/rules.py`, `lulo_app/core/engine.py`, `lulo_app/core/contact.py`, `lulo_app/api_v3.py`, `lulo_app/templates/app.html`, `public/static/v3/app.js`, `public/static/v3/screens.js`, `public/static/v3/app.css`, `public/static/img/` (5 imágenes nuevas), `tests/test_ml.py`, `tests/test_api_v3.py` (versión v1.2), `requirements-dev.txt`, `docs/MODELOS.md`, `README.md`, `CLAUDE.md`.
- Cómo se verificó: `python -m pytest` → 67 passed; `python -m lulo_app.core backtest` → 0 violaciones; `python scripts/train_models.py --check` → modelos.json coincide con el entrenamiento; recorrido de la app con Playwright sin errores de consola.
- Consultado con SOKA: sí; Julián confirmó su aprobación el 2026-10-16.
- Cifras nuevas y su origen: todas salen de `scripts/train_models.py` sobre la base (sha256 5029790b7a29…) y quedan en `lulo_app/ml/modelos.json`; resumen en `docs/MODELOS.md`. El modelo cambia el ramo ofrecido a 50 de 600 clientes en el plan de campaña al corte (cálculo: `engine.campaign()` con y sin `rules.ML_PROPENSITY`).
- Modelo de Claude usado: claude-opus-5-5.

## 2026-10-14 · Claude con SOKA · rama main · versión 3 en producción
- Qué cambió: la versión 3 pasó a `main` (avance rápido desde `04f1b26`, sin reescribir historia) y quedó en producción en https://prototipo-iind.vercel.app. Antes se creó la rama `backup-main` en `04f1b26`, el `main` anterior, que queda congelada junto con `v2-cliente`. `CLAUDE.md`, `README.md` y `.claude/settings.json` se ajustaron al nuevo flujo: ramas `feat/<nombre>-<tema>` desde `main` y pull request hacia `main` que aprueba SOKA. Además, se bloquea el push a `backup-main`.
- Por qué: SOKA pidió que todo el equipo trabaje sobre la versión más completa y que las reglas de colaboración estén en `main`.
- Archivos: `CLAUDE.md` (secciones 3, 6 y 7), `README.md`, `.claude/settings.json`, `CAMBIOS.md`.
- Cómo se verificó: `python -m pytest` → 58 passed; `python -m lulo_app.core backtest` → 0 violaciones; `04f1b26` es ancestro de la rama `v3` (avance rápido posible); el Preview de Vercel del commit quedó en `success` antes de mover `main`.
- Consultado con SOKA: sí, lo pidió el 2026-10-14.
- Cifras nuevas y su origen: ninguna.
- Modelo de Claude usado: claude-opus-5-5.

## 2026-10-14 · Claude con SOKA · rama v3 · commit 5c928da
- Qué cambió: reglas de colaboración para compartir el repo con el equipo. `CLAUDE.md` con el modelo permitido, el alcance, qué es un cambio grande, la documentación obligatoria, las invariantes, el plan y el flujo de git. Además `.claude/settings.json`, que fija el modelo `best` y bloquea push a `main` y `v2-cliente`, push forzado y la edición manual de la base. También este registro, `.github/CODEOWNERS` y la plantilla de pull request.
- Por qué: SOKA va a compartir el repo con compañeros y pidió que sus instancias de Claude sigan solo lo planeado, usen el modelo más capaz, consulten los cambios grandes y documenten todo.
- Archivos: `CLAUDE.md`, `CAMBIOS.md`, `.claude/settings.json`, `.github/CODEOWNERS`, `.github/pull_request_template.md`, `README.md`.
- Cómo se verificó: `python -m pytest` → 58 passed; `.claude/settings.json` es JSON válido.
- Consultado con SOKA: sí, lo pidió el 2026-10-14.
- Cifras nuevas y su origen: ninguna.
- Modelo de Claude usado: claude-opus-5-5.

## 2026-10-10 · Claude con SOKA · rama v3 · commits 118b1d1 a 0db4181
Construcción de la versión 3, por etapas, con aprobación de SOKA en cada punto de control.

- **118b1d1 · E1 Ingesta.** Se agregaron `scripts/ingest_reto.py` y `lulo_app/dataset/` (CSV, `manifest.json`, `CALIDAD.md`). Verificación: 33 passed y reingesta idéntica byte a byte. Cifras: 444 movimientos y 118 abonos posteriores al corte excluidos; `conversion_impactados` = ventas/elegibles en 5 de 5 filas (Campanas_Previas).
- **66a483d · E2 Motor v3.** Se agregó `lulo_app/core` con reglas v1 aprobadas por SOKA: tabla de cobertura, valor esperado, control 10 %, banner al tope y 56+ excluido. Verificación: 50 passed y backtest con 0 violaciones.
- **bc80632 · Reglas v1.1.** Fecha límite primero, 7 días entre push y próximos disparos, aprobados por SOKA. Verificación: 52 passed y backtest con 0 violaciones.
- **2451b01 · E3 API v3.** Se agregaron `/api/v3` y Tu año calculado con el motor. Verificación: 57 passed. Cifra: Sebastián conserva $33 M (suma de ingresos de los 365 días del periodo).
- **8b76534 · E4 App v3.** Look de la app real; `/` sirve v3 y `/v2` conserva la versión del cliente. Verificación: 58 passed, la demo guiada de 16 pasos corre sin errores de consola y el Preview de Vercel quedó en `success`. Los colores salen de capturas de la app real.
- **0db4181 · Ajustes.** Tipografía y densidad como la app real; coberturas en el catálogo. Verificación: 58 passed y Preview en `success`.
- Modelo de Claude usado: claude-opus-5-5.
