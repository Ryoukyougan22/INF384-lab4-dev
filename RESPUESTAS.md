
Alumno: [20210477]
Presupuesto: [13] min  Métrica DORA: [time to restore]



## Parte 1 · Diagnóstico

### Defecto 1
- Job y etapa: smoke, etapa post-deploy.
- Problema que causa en el pipeline: El job smoke depende de build-push en lugar de deploy, permitiendo que se ejecute antes de finalizar el despliegue en AWS Lambda. Esto puede provocar que se valide una versión anterior de la aplicación.
- Link al commit: https://github.com/Ryoukyougan22/INF384-lab4-dev/commit/67fe4e0efae407bc0b7c9ada134930a19fc5df47
- Corrección (una o más: mover de etapa / dejar como informativo / eliminar / ajustar configuración / modificar tests): Ajustar configuración / mover de etapa. Cambiar needs: build-push por needs: deploy en el job smoke.
- Evidencia (dato observado, con link al run o fragmento de la salida local): En la ejecución inicial ci #1, el gráfico muestra que deploy (22 s) y smoke (28 s) dependen directamente de build-push, sin una dependencia entre ellos. Esto evidencia que no se garantiza que el smoke test espere a que termine el despliegue.

### Defecto 2
- Job y etapa: smoke, etapa post-deploy.
- Problema que causa en el pipeline: El script smoke.sh permite comprobar el commit desplegado mediante EXPECTED_COMMIT, pero el workflow no proporcionaba esta variable. Por tanto, la comprobación de la versión se omitía y el smoke test podía aprobar una versión diferente de la que se acababa de construir.
- Link al commit: https://github.com/Ryoukyougan22/INF384-lab4-dev/commit/fe91df0ce8303b522b47444e5d7bebcb4083146a
- Corrección (una o más: mover de etapa / dejar como informativo / eliminar / ajustar configuración / modificar tests): Ajustar configuración. Incorporar EXPECTED_COMMIT: ${{ github.sha }} en el entorno del paso Smoke para obligar a comprobar el commit desplegado.
- Evidencia (dato observado, con link al run o fragmento de la salida local): En la ejecución inicial ci #1, el job smoke finalizó sin mostrar la línea independiente de validación del commit. Después de la corrección, en la ejecución ci #3, el job smoke terminó exitosamente en 10 segundos y mostró:
salud: {"ok":true,"version":"1.0.0","commit":"fe91df0ce8303b522b47444e5d7bebcb4083146a"}
commit: fe91df0ce8303b522b47444e5d7bebcb4083146a

### Defecto 3
- Job y etapa: e2e y load, etapa post-deploy.
- Problema que causa en el pipeline: Los dos jobs dependen únicamente de smoke, por lo que pueden ejecutarse en paralelo sobre la misma función Lambda. Como las pruebas realizan transferencias que modifican saldos, la carga puede interferir con las comprobaciones E2E, provocando resultados inestables.
- Link al commit: https://github.com/Ryoukyougan22/INF384-lab4-dev/commit/ef15442cf1f2950fed9cd9ba1e155b0d8ce2d3b2
- Corrección (una o más: mover de etapa / dejar como informativo / eliminar / ajustar configuración / modificar tests): Ajustar configuración. Hacer que load dependa de e2e para evitar que las pruebas de carga y las pruebas funcionales se ejecuten simultáneamente.
- Evidencia (dato observado, con link al run o fragmento de la salida local): En la ejecución inicial ci #1, el gráfico mostraba que e2e (43 s) y load (12 min 10 s) dependían directamente de smoke, permitiendo su ejecución paralela.
Después de la corrección, en la ejecución ci #4, el gráfico muestra la secuencia smoke → e2e → load. El job smoke finalizó en 7 segundos y e2e en 40 segundos. Solo después comenzó load, que se encontraba en ejecución al momento de la captura. Esto confirma que ambos jobs ya no se ejecutan simultáneamente dentro del mismo run.

### Defecto 4
- Job y etapa: load, etapa post-deploy.
- Problema que causa en el pipeline: La prueba de carga considera aceptables las respuestas HTTP 200 y 400 para las transferencias. Esto permite que una transferencia rechazada por saldo insuficiente se contabilice como una comprobación exitosa. Por tanto, los indicadores de error pueden ocultar fallos de las transferencias que el escenario pretende completar.
- Link al commit: https://github.com/Ryoukyougan22/INF384-lab4-dev/commit/c871d3ca8db278a17c9155dbad4d9e6e4dc1c8fb
- Corrección (una o más: mover de etapa / dejar como informativo / eliminar / ajustar configuración / modificar tests): Modificar tests y ajustar configuración. Diferenciar las transferencias exitosas de las rechazadas y establecer comprobaciones que permitan medir correctamente la tasa de éxito de las operaciones.
- Evidencia (dato observado, con link al run o fragmento de la salida local): En la ejecución original ci #1, k6 registró 34 412 solicitudes HTTP, 100 % de checks exitosos y http_req_failed: 0.00%, utilizando la comprobación transferir 200 o 400.
Después de la corrección, en ci #5, k6 completó 12 minutos de prueba con 30 usuarios virtuales máximos y registró los siguientes resultados:
checks
- 'rate>0.99' rate=100.00%
http_req_duration
- 'p(95)<500' p(95)=82.34ms
http_req_failed
- 'rate<0.01' rate=0.00%
checks_succeeded: 100.00% 34412 out of 34412
checks_failed:      0.00% 0 out of 34412
- transferir 200
- saldo 200
Los resultados confirman que el escenario corregido utiliza la comprobación transferir 200 y aprobó todos los umbrales. No se observaron rechazos HTTP 400 en esta ejecución.

### Defecto 5
- Job y etapa: Pipeline completo, especialmente deploy y las pruebas post-deploy.
- Problema que causa en el pipeline: La exclusión de ejecuciones concurrentes se aplica únicamente al job deploy. Esto evita que dos despliegues ocurran simultáneamente, pero no impide que las pruebas post-deploy de una ejecución coincidan con el despliegue de otra. Como todos los PR comparten la misma función Lambda, una ejecución puede terminar validando una versión que no le corresponde.
- Link al commit: https://github.com/Ryoukyougan22/INF384-lab4-dev/commit/8acb57fce9b9b86411a05eb865be76bde2250c76
- Corrección (una o más: mover de etapa / dejar como informativo / eliminar / ajustar configuración / modificar tests): Ajustar configuración. Agregar una política de concurrency a nivel del workflow completo, utilizando un grupo por repositorio y cancel-in-progress: false, sin modificar el job deploy.
- Evidencia (dato observado, con link al run o fragmento de la salida local):  En la configuración original de ci.yml, líneas 175–177, concurrency estaba definido únicamente dentro del job deploy. Después de la corrección, el commit 8acb57f muestra la incorporación de las siguientes líneas a nivel global:
concurrency:
  group: pipeline-${{ github.repository }}
  cancel-in-progress: false
En la ejecución ci #6, correspondiente al commit 8acb57f, se observa que los jobs unit, integration, sast, sca, build-push, deploy, smoke, e2e y load finalizaron correctamente. Enlace del run ci #6: https://github.com/Ryoukyougan22/INF384-lab4-dev/actions/runs/40963836648


## Parte 2 · Implementación
- Link al run en verde en main: https://github.com/Ryoukyougan22/INF384-lab4-dev/actions/runs/40963836648
- Link al pull request: https://github.com/Ryoukyougan22/INF384-lab4-dev/pull/1
- Defecto de la aplicación introducido (archivo y descripción): Se modificó el archivo src/util.js, dentro de la función validarMonto, reemplazando la condición monto <= 0 por monto < 0. Esto permite incorrectamente transferencias con monto cero, incumpliendo la regla de negocio que exige que los montos sean estrictamente positivos.
- Job que debería detenerlo: unit, porque ejecuta las pruebas unitarias que verifican la validación de montos. En particular, la prueba validarMonto rechaza cero comprueba que se rechacen los montos iguales a cero.
- Link al run fallido: https://github.com/Ryoukyougan22/INF384-lab4-dev/actions/runs/40965928536/job/123770296469?pr=1
- Resultado observado: El job unit falló durante la ejecución del pull request, mostrando not ok 4 - validarMonto rechaza cero. La prueba esperaba el mensaje monto debe ser mayor que cero, pero obtuvo null, lo que evidencia que la aplicación aceptaba incorrectamente el monto cero. Como consecuencia, los jobs build-push, deploy, smoke, e2e y load fueron omitidos por sus dependencias. Esto demuestra que el pipeline detectó el defecto antes de construir y desplegar la aplicación.

## Parte 3 · Decisión bajo restricción
Link al run medido: https://github.com/Ryoukyougan22/INF384-lab4-dev/actions/runs/40963836648

| Job | Duración (min:seg) |
|---|---|
| unit | 0:10 |
| integration | 0:16 |
| sast | 0:24 |
| sca | 0:15 |
| build-push | 0:47 |
| deploy | 0:19 |
| smoke | 0:11 |
| e2e | 0:43 |
| load | 12:09 |
| **Total** | 14:45 |

Resultado de load en ese run:
| Métrica | Valor |
|---|---|
| p90 | 16.98ms |
| p95 | 19.33 ms |
| Máximo | 183.92 ms |
| Tasa de error | 0.00 % |
| Requests por segundo | 53.60 req/s |

Conclusión sobre el comportamiento bajo carga: Durante la ejecución ci #6, Monedero Andino procesó 38 612 solicitudes HTTP durante 12 minutos, alcanzando un máximo de 30 usuarios virtuales. Se obtuvo una latencia p95 de 19.33 ms, una latencia máxima de 183.92 ms y una tasa de error del 0 %. Todas las comprobaciones resultaron exitosas. Estos resultados evidencian un rendimiento satisfactorio bajo las condiciones evaluadas. Sin embargo, no permiten garantizar el mismo comportamiento ante cargas superiores ni la consistencia de los saldos entre distintas instancias AWS Lambda, debido al uso de SQLite local en /tmp.

SLO propuesto (percentil, valor, ventana) y justificación: Se propone un SLO de latencia HTTP p95 inferior a 150 ms, medido durante una prueba de carga de 8 minutos con hasta 30 usuarios virtuales, incluyendo 6 minutos de carga constante. Se establece mediante SLO_P95: 150 en ci.yml. El valor presenta un margen respecto del p95 observado de 19.33 ms en ci #6 y de 82.34 ms en ci #5. Esto permite contemplar variaciones normales entre ejecuciones y mantiene una exigencia mayor que el umbral original de 500 ms.

SLA propuesto y diferencia con el SLO: Se propone un SLA de latencia HTTP p95 inferior a 300 ms, medido mensualmente sobre las solicitudes de la API, como compromiso hipotético con los clientes. Su incumplimiento podría contemplar compensaciones según las condiciones contractuales. El SLO es una meta técnica interna, mientras que el SLA constituye un compromiso formal con los clientes. Se propone un SLA menos exigente para mantener margen operativo y evitar convertir fluctuaciones menores en incumplimientos contractuales.

Cambios para cumplir el presupuesto:
| Cambio | Job | Minutos recuperados |
|---|---|---|
|Reducir la prueba de carga de 12 a 8 minutos, manteniendo 6 minutos de carga constante | load | 4:00 estimados |
|Reducir SLO_P95 de 500 a 150 ms | load | 0:00 |
|Mantener SAST obligatorio en pre-build | sast | 0:00 |

Total resultante: [ ] min
Link al run en main con los cambios:

Riesgo aceptado (defecto de la aplicación concreto y efecto en la métrica DORA asignada):

Al reducir la prueba de carga de 12 a 8 minutos, se acepta el riesgo de no detectar defectos de rendimiento que se manifiesten únicamente después de periodos prolongados de uso. Por ejemplo, una fuga de recursos introducida en el procesamiento de transferencias de src/wallet.js podría provocar errores HTTP 500 después de nueve minutos de carga sostenida. El pipeline original podría detectar ese problema, mientras que el optimizado terminaría antes de que se manifieste. Respecto a la métrica DORA Time to Restore, la reducción permite validar más rápidamente los cambios destinados a recuperar el servicio. Sin embargo, también aumenta el riesgo de declarar una recuperación satisfactoria sin haber detectado un problema tardío, lo que podría provocar nuevos incidentes y prolongar el tiempo real de recuperación.

Decisión sobre sast (se queda / se mueve a otra etapa o evento / informativo) y razón: 

Se decide mantener SAST obligatorio en pre-build. En ci #6 su duración fue de solo 24 segundos, por lo que el ahorro potencial de eliminarlo es reducido en comparación con el riesgo de permitir el despliegue de código con vulnerabilidades detectables mediante análisis estático. Mantener esta comprobación favorece una recuperación más confiable, evitando que una corrección urgente introduzca nuevos problemas de seguridad que puedan extender el Time to Restore.

## Declaración de uso de IA
Herramientas usadas y para qué: Se utilizó ChatGPT (OpenAI) como herramienta de apoyo para comprender los conceptos de CI/CD, GitHub Actions, AWS Lambda y pruebas automatizadas; analizar los defectos del pipeline; proponer y explicar las correcciones; interpretar los resultados de los logs y las pruebas de carga con k6; y apoyar la redacción y justificación de las respuestas del laboratorio. Las modificaciones del repositorio, la ejecución de los workflows y la recopilación de evidencias se realizaron mediante GitHub y AWS Academy.
