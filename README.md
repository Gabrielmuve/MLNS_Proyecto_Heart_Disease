# Estratificación de riesgo coronario: qué pruebas del tamizaje aportan información

Análisis probabilístico de 297 pacientes de la base de Cleveland (UCI Heart Disease), hecho desde
el rol de analista de datos de una red de clínicas cardiológicas.

Proyecto de Primer Corte, Aprendizaje de Máquina No Supervisado, Universidad de La Sabana, 2026-II.

## El problema de negocio

De cada 100 pacientes que llegan a la red derivados por sospecha coronaria, 46 tienen la
enfermedad. Antes de aplicar cualquier prueba, la incertidumbre diagnóstica es de 0.996 bits sobre
un máximo posible de 1.0, prácticamente la de lanzar una moneda.

La red aplica un panel de exámenes de rutina sin evidencia cuantitativa de cuánto reduce cada uno
esa incertidumbre. La pregunta del proyecto es exactamente esa: qué pruebas informan el diagnóstico
y cuáles no.

## El dataset

Heart Disease Dataset, base de Cleveland, del UCI Machine Learning Repository.

Janosi, A., Steinbrunn, W., Pfisterer, M. y Detrano, R. (1989). *Heart disease* [Data set].
UCI Machine Learning Repository. https://doi.org/10.24432/C52P4X · Licencia CC BY 4.0

303 pacientes y 14 atributos clínicos. Se descarga con el paquete oficial `ucimlrepo`:

```python
from ucimlrepo import fetch_ucirepo
heart_disease = fetch_ucirepo(id=45)
```

Se eliminaron 6 registros con valores faltantes en `ca` o `thal`, quedando n = 297. La variable
objetivo `num`, que va de 0 a 4 según severidad, se binarizó como enfermedad presente cuando
`num >= 1`, porque la pregunta de negocio es si derivar o no al paciente, no qué tan grave está.

## Hallazgos

**Dos exámenes de rutina no aportan información diagnóstica en esta población.**
El colesterol tiene una divergencia KL de 0.080 bits entre enfermos y sanos, 17.8 veces menor que
la de la depresión del segmento ST. Sus medias no se diferencian significativamente (251.9 vs
243.5 mg/dl, p = 0.165) ni sus varianzas (Levene p = 0.721). La glucemia en ayunas resultó
directamente independiente del diagnóstico: chi-cuadrado de 0.00 con p = 1.000.

**El paciente asintomático es el de mayor riesgo.**
Quienes no reportan dolor típico de angina tienen 72.5% de enfermedad, frente al 46.1% de la base
general y al 21.9% de los demás tipos de dolor. Descartar por ausencia de angina típica invierte el
orden correcto del triage.

**Tres pruebas en secuencia llevan la sospecha de 46.1% a 97.7%.**
El resultado final es idéntico en las seis permutaciones del orden, porque en odds la actualización
bayesiana es una multiplicación. Eso permite ordenar las pruebas por costo sin perder precisión.

**Un modelo multivariado sí aporta señal real.**
La regresión logística baja la entropía cruzada de 0.691 nats de la línea base a 0.324 en datos que
no vio, con AUC de 0.938 y sin sobreajuste.

## Los 11 conceptos y dónde están

| # | Concepto | Cifra clave |
|---|---|---|
| 1 | Probabilidad condicional | P(D \| asintomático) = 72.5% frente a P(D) = 46.1% |
| 2 | Teorema de Bayes | P(D \| angina por ejercicio) = 76.3%, LR+ = 3.76 |
| 3 | Verosimilitud | Comparación de log-verosimilitud Normal contra Log-Normal |
| 4 | Máxima verosimilitud (MLE) | Colesterol: 251.9 vs 243.5 mg/dl |
| 5 | Distribuciones paramétricas | Bernoulli, Categórica y Gaussiana; ΔAIC = 29.9 a favor de Log-Normal en colesterol |
| 6 | Esperanza y varianza | Var(thalach) = 515.8 en enfermos contra 362.7 en sanos, Levene p = 0.014 |
| 7 | Independencia y correlación | Sexo: chi² = 21.85, V de Cramér = 0.271. Glucemia: chi² = 0.00 |
| 8 | Prior y posterior | 46.1% a 97.7%, invariante al orden de las pruebas |
| 9 | Entropía | H(D) = 0.996 bits, H(D \| dolor) = 0.799, ganancia de 0.197 |
| 10 | Entropía cruzada | 0.324 nats en prueba contra 0.691 de la línea base |
| 11 | Divergencia KL | Colesterol 0.080 bits contra oldpeak 1.423 |

## Recomendaciones

1. Adelantar la prueba de esfuerzo por delante del panel lipídico en la ruta de derivación. Las dos
   variables más informativas, depresión del ST y frecuencia cardíaca máxima, salen del mismo examen.
2. Retirar la ausencia de angina típica como criterio de descarte en el triage.
3. Aplicar las pruebas en orden ascendente de costo y detenerse al cruzar el umbral de decisión.
4. Recalibrar el prior y validar el modelo sobre la población propia antes de llevarlo a producción.

## Estructura del repositorio

```
.
├── data/
│   └── heart_cleveland.csv
├── notebooks/
│   └── Proyecto_Primer_Corte_Heart_Disease.ipynb
├── informe/
│   └── Informe_Gerencial_Heart_Disease.pdf
└── README.md
```

## Cómo reproducirlo

```bash
pip install numpy pandas scipy scikit-learn matplotlib jupyter ucimlrepo
jupyter notebook notebooks/Proyecto_Primer_Corte_Heart_Disease.ipynb
```

Ejecutar de principio a fin con Restart & Run All. La semilla está fija en 42, así que todas las
cifras del informe se reproducen exactamente.

## Limitaciones

La muestra es de 297 pacientes de un solo centro, recolectados en 1988, así que los perfiles de
riesgo y la práctica clínica han cambiado y los intervalos de confianza son amplios.

La actualización bayesiana secuencial supone independencia condicional entre pruebas. El contraste
empírico muestra que ese supuesto sobreestima el posterior en unos 3 puntos: 97.7% teórico frente a
94.3% observado en los 35 pacientes que presentan las tres señales. Las tres pruebas miden aspectos
relacionados de la misma isquemia.

La prevalencia de 46.1% corresponde a pacientes ya derivados por sospecha y no a tamizaje abierto,
por lo que el prior debe recalibrarse antes de aplicar estos posteriores a otra población.

Todos los resultados son asociaciones observacionales, no relaciones causales.
