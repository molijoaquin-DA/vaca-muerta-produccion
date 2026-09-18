# Vaca Muerta: priorización de pozos para revisión productiva

## Problema

Los equipos de producción tienen una capacidad limitada para revisar
pozos. Analizar su comportamiento histórico puede ayudar a identificar
casos con riesgo de caída productiva y orientar esa revisión.

## Objetivo

Analizar la producción histórica de pozos operados por YPF en Vaca
Muerta y evaluar si modelos de aprendizaje automático permiten
priorizar pozos con riesgo de caída de producción.

La pregunta principal es:
¿podemos mejorar la selección de pozos frente a reglas sencillas,
manteniendo la misma cantidad de revisiones mensuales?

## Fuente de datos

Se utilizarán datos públicos de la Secretaría de Energía de Argentina,
correspondientes a producción de petróleo y gas por pozo — Capítulo IV.

El punto de partida será el archivo de producción no convencional,
conservado sin modificaciones.

- Catálogo:
  https://datos.energia.gob.ar/dataset/produccion-de-petroleo-y-gas-por-pozo
- Documentación de referencia:
  https://github.com/datosenergia/produccion-de-petroleo-y-gas-por-pozo

Las instrucciones de descarga y la identificación de la versión
utilizada se incorporarán al preparar los datos.

## Alcance inicial

Se explorará la base original para justificar la selección de empresa,
formación, áreas y período de análisis.

Loma Campana, La Amarga Chica y Bandurria Sur son áreas propuestas;
su inclusión dependerá de la calidad y cobertura de los datos.

## Etapas previstas

1. Comprensión del problema y de los datos.
2. Análisis exploratorio.
3. Limpieza y preparación, documentando cada decisión.
4. Construcción de variables a partir de información histórica.
5. Regresión para estimar producción.
6. Clasificación para identificar riesgo de caída productiva.
7. Clustering para explorar perfiles de pozos.
8. Evaluación temporal y comparación con reglas sencillas.
9. Comunicación de resultados y limitaciones.

## Evaluación de utilidad

Se simulará una capacidad limitada de revisión mensual y se comparará
cuántas caídas relevantes identifica cada método.

La definición de caída relevante y la cantidad de pozos a revisar
se establecerán y justificarán durante el análisis.

## Limitaciones

Una caída de producción no implica necesariamente una falla ni un
volumen recuperable.

Se considerarán las demoras de publicación de los datos para evitar
utilizar información que no habría estado disponible al emitir
una predicción.

Este es un proyecto académico con datos públicos. No constituye una
herramienta operativa validada ni representa una posición oficial de YPF.

## Estado del proyecto

Etapa inicial: organización del repositorio y documentación del problema.
El análisis se desarrollará paso a paso a partir de los datos crudos.
