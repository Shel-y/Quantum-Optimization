# Proyecto QUBO-QAOA: Matching Bipartito 4×4 para Encharcamientos en la CDMX

**Autores:**
- Integrante 1: `Joselyn Montserrat Lagunas Sanabria` – `Universidad Virtual del Estado de Guanajuato (UVEG)`
- Integrante 2: `Angel Rodrigo Quintal Vega` – `Tecnológico Nacional de México (TecNM) Campus Mérida - Instituto Tecnológico de Mérida (ITM)`

**Curso:** QMexico Summer School 2026  
**Repositorio:** `https://github.com/Shel-y/Quantum-Optimization.git`

---

## 1. Dataset y Justificación 

| **Criterio** | **Descripción y Cumplimiento** |
| :--- | :--- |
| **Nombre del dataset** | Reportes de agua de la CDMX (SACMEX) - Filtrado por "Encharcamiento". |
| **Fuente oficial** | Sistema de Aguas de la Ciudad de México (SACMEX). |
| **Institución responsable** | Gobierno de la Ciudad de México. |
| **URL de la fuente** | Los datos se obtuvieron a través del portal de datos abiertos de la CDMX - `https://datos.cdmx.gob.mx/dataset/reportes-de-agua/resource/a8069e94-c7cb-45d7-8166-561e80884422` |
| **URL raw del CSV usado en `data/`** | `https://raw.githubusercontent.com/Shel-y/Quantum-Optimization/main/data/dataset_real_4x4.csv` |
| **Licencia o condiciones de uso** | Datos públicos abiertos|
| **Fecha de consulta** | `22 de Junio de 2026` |
| **Dominio del problema** | Gestión de infraestructura hídrica y atención de encharcamientos en la CDMX. |

### Justificación del Modelado
El problema se modela como un **matching bipartito** porque buscamos asignar 4 puntos críticos (colonias con alta incidencia de encharcamientos) a 4 corredores viales (rutas de atención) de manera uno-a-uno, lo cual se busca optimizar y llegar a una asignación exclusiva y unitaria (Usuario - Ruta) tras ejecutar los algoritmos cuánticos y clásicos. 

**Conjunto A (Usuarios/Orígenes):** Las 4 colonias con mayor número de reportes de encharcamiento, que representan los focos de atención.
> *Criterio de selección:* Posterior al depuramiento de los datos de la SACMEX, se agruparon las coordenadas geográficas (redondeadas a 3 decimales) de los reportes de encharcamiento de 2022-2023, se contó la frecuencia por coordenada y se seleccionaron las 4 más recurrentes.

**Conjunto B (Recursos/Rutas):** 4 vialidades primarias de la CDMX, elegidas por ser corredores viales estratégicos donde se concentran problemas de drenaje.
> *Criterio de selección:* Vialidades relevantes en el mapa de la ciudad (Viaducto, Circuito Interior, Periférico y Eje Central).

**Definición de $$x_{ij}=1$$** Significa asignar la cuadrilla o atención prioritaria del punto crítico `i` a la vialidad `j`.  
**Definición de $$x_{ij}=0$$** No se asigna ese recurso a esa vialidad.

---

## 2. Matriz de Score (Compatibilidad \( S \))

Para construir una matriz **no degenerada** que refleje una interacción real entre el origen (colonia) y el destino (vialidad), se utilizó la **distancia geográfica**.

### Columnas usadas y Fórmula
1. **Cálculo de distancia:** Se calcularon las distancias en metros entre cada punto crítico (coordenadas `lat_i` y `lon_i`) y cada ruta vial (representada como una polilínea de *waypoints*).

2. **Distancia de Haversine:** Se utilizó la fórmula de Haversine para calcular la distancia geodésica entre dos puntos sobre la superficie terrestre.

3. **Normalización inversa:** Las distancias se transformaron en un puntaje de compatibilidad mediante una normalización min-max invertida:

```text
S(i,j) = 1 - (d(i,j) - min(d)) / (max(d) - min(d))
```

Donde:

- `d(i,j)` es la distancia mínima (en metros) entre el punto crítico `i` y la ruta `j`.
- `min(d)` y `max(d)` corresponden a la distancia mínima y máxima observadas en toda la matriz de distancias.

**Interpretación del score:** Un valor cercano a **1.0** indica que el punto crítico se encuentra muy próximo a la ruta (mayor compatibilidad y prioridad de atención), mientras que un valor cercano a **0.0** indica una menor compatibilidad debido a una mayor distancia.

### Matriz \( S \) 4×4 resultante (obtenida experimentalmente)

| A \ B | **R1 (Viaducto)** | **R2 (Circuito)** | **R3 (Periférico)** | **R4 (Eje Central)** |
| :--- | :---: | :---: | :---: | :---: |
| **U1 (Barrio San Miguel)** | 0.50 | 0.61 | 0.52 | **0.66** |
| **U2 (Colinas Del Ajusco)** | 0.31 | 0.00 | **0.72** | 0.01 |
| **U3 (Jardines Del Pedregal)** | **0.63** | 0.31 | 1.00 | 0.32 |
| **U4 (La Joya)** | 0.38 | **0.16** | 0.80 | 0.19 |

---

## 3. Restricciones y Formulación QUBO

El problema de asignación se modela como un problema de *matching* bipartito uno-a-uno con las siguientes restricciones:

- **Restricción por filas:** Cada colonia debe asignarse a exactamente una ruta.

```text
Σⱼ x(i,j) = 1    para toda colonia i
```

- **Restricción por columnas:** Cada ruta debe recibir exactamente una colonia.

```text
Σᵢ x(i,j) = 1    para toda ruta j
```

### Justificación del modelo QUBO

El problema se transforma en un modelo **QUBO (Quadratic Unconstrained Binary Optimization)** para poder resolverlo mediante algoritmos cuánticos como **QAOA**.

La función de energía utilizada es:

```text
E(x) =
- Σ S(i,j) · x(i,j)
+ λA Σ (Σ x(i,j) - 1)²
+ λB Σ (Σ x(i,j) - 1)²
```

donde:

- `S(i,j)` representa el score de compatibilidad entre la colonia `i` y la ruta `j`.
- `x(i,j)` es una variable binaria que vale **1** si la asignación se realiza y **0** en caso contrario.
- **λA = λB = 5**, un valor mayor que el score máximo posible, lo que garantiza que cualquier violación de las restricciones tenga una penalización superior al beneficio obtenido por maximizar el score.

Este es un modelo estándar ampliamente utilizado en problemas de optimización combinatoria formulados como QUBO.
---

## 4. Resultados

### Solución Clásica Exacta (Fuerza Bruta sobre $2^{16}$ estados)
La fuerza bruta encontró la asignación óptima con las siguientes características:
- **Mejor asignación:** `U1-R4, U2-R3, U3-R1, U4-R2`.
- **Score total:** `2.17`.
- **Energía QUBO:** `-2.17`.
- **¿Cumple restricciones?** Sí.

### Resultado QAOA Local (Simulación $p=1$, COBYLA)
- **Parámetros óptimos:** $$ \gamma \approx `-1.7674`, \beta \approx `0.4382` $$.
- **Energía esperada:** `17.343`.
- **Brecha (energía esperada vs. óptimo clásico):** `19.513`.
- **Mejor muestra observada (2000 shots):** `U1-R4, U2-R3, U3-R1, U4-R2` (Energía `-2.17`, Score `2.17`).
- **Probabilidad de factibilidad (distribución ideal):** `2.34%`.
- **Probabilidad del óptimo clásico específico (distribución ideal):** `0.099%`.

**Hardware real (IBM Quantum):** *No se ejecutó en hardware real para esta entrega base. El pipeline está listo para ser extendido en trabajos futuros.*

---

## 5. Ética y Limitaciones

### Riesgos éticos identificados

- El modelo utiliza datos agregados por colonia, sin incluir nombres, CURP, direcciones específicas ni otra información personal, por lo que no compromete la privacidad individual.
- No se emplea para tomar decisiones reales de asignación de recursos; su propósito es exclusivamente académico y demostrativo.

### Medidas de mitigación

- Se utilizaron únicamente datos públicos provenientes del portal de Datos Abiertos de la Ciudad de México.
- Tanto este README como el notebook aclaran que la salida de QAOA **no constituye una recomendación operativa o normativa** para SACMEX.

### Limitaciones del modelo (4×4)

- La escala del problema es reducida (4 puntos críticos y 4 rutas), por lo que no representa la complejidad real de la red de atención de la Ciudad de México.
- El score de compatibilidad se basa únicamente en la distancia geográfica y no considera variables operativas como capacidad de bombeo, disponibilidad de personal, severidad del encharcamiento o condiciones de tráfico.
- Si el problema creciera, por ejemplo a una instancia de **10×10**, la búsqueda por fuerza bruta sería computacionalmente inviable (2¹⁰⁰ posibles estados), por lo que sería necesario recurrir a algoritmos de optimización más escalables o a hardware cuántico.

### ¿Qué cambiaría si el dataset creciera?

Para instancias de mayor tamaño sería recomendable utilizar un *mixer* que preserve las restricciones (por ejemplo, **XY Mixer**) para incrementar la probabilidad de obtener soluciones factibles. Asimismo, podrían requerirse más capas (**p**) en QAOA y optimizadores clásicos más robustos, como **SPSA**, para mejorar la calidad de las soluciones obtenidas.
---

## 6. Ejecución

**Instrucciones para abrir en Google Colab:**
1. Clonar este repositorio o descargar el archivo `qubo_qaoa_encharcamientos.ipynb`.
2. Subir el archivo a Google Colab (`Archivo -> Subir notebook`).
3. Asegurarse de que el archivo `data/dataset_real_4x4.csv` esté en la raíz del entorno de Colab, o pegar la URL raw en la celda designada para la carga del dataset.
4. Ejecutar todas las celdas en orden (`Runtime -> Run all`).

**Estado de ejecución:** Todas las celdas corren sin errores intermedios en un entorno estándar de Colab con conexión a internet para la instalación de dependencias.

---

## 7. Interpretación Mínima del Resultado

1. **¿Cuál fue la mejor asignación encontrada?**  
   `U1-R4, U2-R3, U3-R1, U4-R2` (tanto en el clásico como en la mejor muestra de QAOA).
2. **¿Cuál fue su score en el dominio?** `2.17` puntos.
3. **¿Cumple restricciones?** Sí, es una asignación uno-a-uno perfecta.
4. **¿QAOA local observó el óptimo clásico?** Sí, en al menos una de las 2000 muestras alcanzó la energía `-2.17`.
5. **¿Qué tan frecuente fue observar soluciones factibles?** Baja (`2.34%`) debido al mixer estándar.
6. **¿Qué limitaciones tiene el modelo $ 4\times 4 $?** Escala demasiado pequeña y simplificación de la realidad.
7. **¿Qué cambiaría si creciera?** Se necesitaría hardware cuántico y mejores mixers.
8. **¿Qué riesgos éticos existen y cómo se mitigaron?** Se evitan datos personales y se declara uso educativo.
9. **¿Se usó hardware real?** No en esta entrega.
10. **¿Se usó reparación clásica?** No se aplicó en el pipeline principal.

---

*Este proyecto fue desarrollado con fines educativos durante la QMexico Summer School 2026.*
