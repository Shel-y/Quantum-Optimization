# Proyecto QUBO-QAOA: Matching Bipartito 4×4 para Encharcamientos en la CDMX

**Autores:**
- Integrante 1: `Joselyn Montserrat Lagunas Sanabria` – `Universidad Virtual del Estado de Guanajuato (UVEG)`
- Integrante 2: `Angel Rodrigo Quintal Vega` – `Tecnológico Nacional de México (TecNM) Campus Mérida - Instituto Tecnológico de Mérida (ITM)`

**Curso:** QMexico Summer School 2026  
**Repositorio:** `https://github.com/Shel-y/Quantum-Optimization.git`

---

## 1. Dataset y Justificación (Criterio ~80% de la evaluación)

| **Criterio** | **Descripción y Cumplimiento** |
| :--- | :--- |
| **Nombre del dataset** | Reportes de agua de la CDMX (SACMEX) - Filtrado por "Encharcamiento". |
| **Fuente oficial** | Sistema de Aguas de la Ciudad de México (SACMEX). |
| **Institución responsable** | Gobierno de la Ciudad de México. |
| **URL de la fuente** | Los datos se obtuvieron a través del portal de datos abiertos de la CDMX - `https://datos.cdmx.gob.mx/dataset/reportes-de-agua/resource/a8069e94-c7cb-45d7-8166-561e80884422` |
| **URL raw del CSV usado en `data/`** | `https://raw.githubusercontent.com/Shel-y/Quantum-Optimization/main/data/dataset_real_4x4.csv` |
| **Licencia o condiciones de uso** | Datos públicos abiertos (sin restricciones para uso académico). |
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
1. **Cálculo de distancia:** Se calcularon las distancias en metros entre cada punto crítico (coordenadas \( lat_i - Latitud i-ésima, lon_i - Longitud i-ésima \)) y cada ruta vial (definida como una polilínea de waypoints).
2. **Fórmula de Haversine:** Para medir la distancia esférica entre dos puntos geográficos.
3. **Normalización inversa:** Convertimos distancia en compatibilidad mediante una normalización min-max invertida:

   $$ S_{ij} = 1 - \frac{d_{ij} - \min(d)}{\max(d) - \min(d)} $$

   Donde:
   - $ d_{ij} $ es la distancia mínima (en metros) desde el punto crítico $ i $ hasta la ruta $ j $.
   - $ min(d) $ y $ max(d) $ son los valores mínimo y máximo de todas las distancias calculadas en la matriz.
   
   **Interpretación del score:** Un valor de `1.0` significa que el punto está muy cerca de la ruta (muy compatible, prioridad alta), mientras que `0.0` significa que está muy lejos (poco compatible, prioridad baja).

### Matriz \( S \) 4×4 resultante (obtenida experimentalmente)

| A \ B | **R1 (Viaducto)** | **R2 (Circuito)** | **R3 (Periférico)** | **R4 (Eje Central)** |
| :--- | :---: | :---: | :---: | :---: |
| **U1 (Barrio San Miguel)** | 0.50 | 0.61 | 0.52 | **0.66** |
| **U2 (Colinas Del Ajusco)** | 0.31 | 0.00 | **0.72** | 0.01 |
| **U3 (Jardines Del Pedregal)** | **0.63** | 0.31 | 1.00 | 0.32 |
| **U4 (La Joya)** | 0.38 | **0.16** | 0.80 | 0.19 |

---

## 3. Restricciones y Formulación QUBO

El problema de asignación se formaliza como:

- **Restricción por filas:** Cada colonia debe ser asignada a exactamente una ruta.
  $$
  \sum_{j=1}^{4} x_{ij} = 1 \quad \forall i \in \{1,\dots,4\}
  $$
- **Restricción por columnas:** Cada ruta debe recibir exactamente una colonia.
  $$
  \sum_{i=1}^{4} x_{ij} = 1 \quad \forall j \in \{1,\dots,4\}
  $$

### Justificación del modelo QUBO
El problema se transforma en QUBO (Quadratic Unconstrained Binary Optimization) para poder ser resuelto por algoritmos cuánticos como QAOA. La función de energía es:

$$
E(x) = -\sum_{i,j} S_{ij} x_{ij} + \lambda_A \sum_i (\sum_j x_{ij} - 1)^2 + \lambda_B \sum_j (\sum_i x_{ij} - 1)^2
$$

Donde $\lambda = 5$ (mayor que el máximo score posible) garantiza que violar una restricción siempre sea más costoso que cualquier beneficio por score. Este modelo es estándar en la literatura de optimización combinatoria y QUBO.

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

**Riesgos éticos identificados:**
- El modelo utiliza datos agregados por colonia, sin nombres, CURP o direcciones específicas, por lo que no vulnera la privacidad individual.
- No se toman decisiones reales de asignación de recursos; esto es un ejercicio académico.

**Medidas de mitigación:**
- Se usaron exclusivamente datos públicos y agregados.
- El README y el notebook dejan claro que la salida de QAOA **no es una recomendación normativa** para la SACMEX.

**Limitaciones del modelo $ 4 \times 4 $:**
- Escala pequeña (solo 4 puntos críticos), no representativa de toda la CDMX.
- El score basado únicamente en distancia ignora factores como capacidad de bombeo, personal disponible o tipo de encharcamiento.
- Si el dataset creciera (ej. $ 10\times 10 $), la fuerza bruta sería inviable ($ 2^{100} $), y se necesitarían técnicas avanzadas de muestreo o hardware cuántico real.

**¿Qué cambiaría si el dataset creciera?**
Se requeriría un mixer que preserve restricciones (ej. *XY-mixer*) para aumentar la probabilidad de factibilidad, y se necesitarían más capas $ p $ en QAOA para aumentar la calidad de la aproximación, así como optimizadores clásicos más robustos (ej. SPSA).

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
