### Escuela Colombiana de Ingeniería
### Arquitecturas de Software - ARSW

## Escalamiento en Azure con Maquinas Virtuales, Sacale Sets y Service Plan
 
### Dependencias
* Cree una cuenta gratuita dentro de Azure. Para hacerlo puede guiarse de esta [documentación](https://azure.microsoft.com/es-es/free/students/). Al hacerlo usted contará con $100 USD para gastar durante 12 meses.

## Integrantes: 
1. Juan Camilo Torres Suarez
2. Valeria Aguilar Bermudez

### Parte 0 - Entendiendo el escenario de calidad

El laboratorio trae una aplicación ya hecha (`FibonacciApp`) que calcula el enésimo número de la secuencia de Fibonacci. Está escrita en Node.js y responde en la ruta `/fibonacci/{n}`.

**Escalabilidad**
Cuando un conjunto de usuarios consulta un enésimo número (superior a 1000000) de la secuencia de Fibonacci de forma concurrente y el sistema se encuentra bajo condiciones normales de operación, todas las peticiones deben ser respondidas y el consumo de CPU del sistema no puede superar el 70%.

En palabras sencillas, para que el sistema "pase la prueba" se tienen que cumplir **dos condiciones al mismo tiempo**:

| Condición | Qué significa | Cómo la medimos |
|---|---|---|
| Todas las peticiones responden | Ningún usuario se queda sin respuesta, aunque varios pregunten a la vez | Columna `failed` de Newman (debe ser 0) |
| CPU ≤ 70 % | La máquina no trabaja al límite | Métrica *Percentage CPU* en Azure Monitor |

| Parte del escenario | En este laboratorio |
|---|---|
| Fuente | Varios usuarios (simulados con Newman) |
| Estímulo | Consultas concurrentes de Fibonacci con n ≥ 1.000.000 |
| Artefacto | La FibonacciApp desplegada en Azure |
| Ambiente | Operación normal |
| Respuesta | Todas las peticiones se responden |
| Medida | 0 peticiones fallidas y CPU ≤ 70 % |

### Parte 1 - Escalabilidad vertical

> **Ajuste importante de tamaños.** El enunciado pide `Standard_B1ls` y luego `Standard_B2ms`, pero la suscripción *Azure for Students* no permite usar la serie B en East US (mensaje `NotAvailableForSubscription`). Tampoco había cuota para las series Falsv7 ni Av2. Por eso usamos la familia **Dsv4**, manteniendo la idea del laboratorio: empezar con una VM y luego **duplicar sus recursos**.
>
> | | Enunciado | Lo que usamos |
> |---|---|---|
> | VM inicial | `B1ls` (1 vCPU, 0,5 GiB) | **`Standard_D2s_v4`** (2 vCPU, 8 GiB) · 0,096 US$/h |
> | VM escalada | `B2ms` (2 vCPU, 8 GiB) | **`Standard_D4s_v4`** (4 vCPU, 16 GiB) · 0,192 US$/h |
>
> ![Serie B no disponible](images/Solutions/part1/01-serie-b-no-disponible.png)

#### 1. Creación de la máquina virtual

Creamos la VM desde el portal de Azure con esta configuración:

* Resource Group = `SCALABILITY_LAB`
* Virtual machine name = `VERTICAL-SCALABILITY`
* Región = East US (Central US estaba bloqueada por la política de la suscripción)
* Image = Ubuntu Server 24.04 LTS (x64, Gen2)
* Size = `Standard_D2s_v4` (ver nota de arriba)
* Username = `scalability_lab`
* Autenticación = llave pública SSH
* Azure Spot desactivado (*Priority: On-demand*), para que Azure no apague la VM en medio de las pruebas.

![Datos básicos 1](images/Solutions/part1/02-vm-datos-basicos-1.png)
![Datos básicos 2](images/Solutions/part1/03-vm-datos-basicos-2.png)

Disco SSD estándar que se borra junto con la VM, y una IP pública nueva con el puerto SSH (22) abierto:

![Discos](images/Solutions/part1/04-vm-discos.png)
![Redes](images/Solutions/part1/05-vm-redes.png)

Activamos el apagado automático a las 9:30 p. m. para no gastar crédito si se nos olvidaba apagarla:

![Apagado automático](images/Solutions/part1/06-vm-apagado-automatico.png)

Resumen antes de crear e implementación terminada:

![Revisar y crear](images/Solutions/part1/07-vm-revisar-y-crear.png)
![Implementación completa](images/Solutions/part1/08-vm-implementacion-completa.png)

Recursos que quedaron dentro del grupo `SCALABILITY_LAB`:

![Recursos del grupo](images/Solutions/part1/09-recursos-del-grupo.png)

#### 2. Regla de entrada para el puerto 3000

La aplicación escucha en el puerto 3000, así que agregamos una regla de entrada en el grupo de seguridad de red (`VERTICAL-SCALABILITY-nsg`): `Port_3000`, prioridad 310, TCP, origen *Any*, acción *Permitir*.

![Regla puerto 3000](images/Solutions/part1/10-regla-puerto-3000.png)

#### 3. Conexión por SSH

`ssh -i "$HOME\.ssh\arsw_lab07" scalability_lab@20.51.178.24`

![Conexión SSH](images/Solutions/part1/11-conexion-ssh.png)

#### 4. Instalación de Node.js con NVM

```
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
source ~/.bashrc
nvm install --lts
```

![Instalación de Node](images/Solutions/part1/12-instalacion-node.png)

#### 5. Instalación y ejecución de la aplicación

```
git clone https://github.com/JuanTorres1303/ARSWLAB07.git
cd ARSWLAB07/FibonacciApp
npm install
npm install -g forever
forever start FibonacciApp.js
```

![Clonar e instalar](images/Solutions/part1/13-clonar-e-instalar-app.png)

En el primer intento `forever list` y `curl` se ejecutaron antes de que la app terminara de arrancar, por eso aparece "No forever processes running":

![Primer intento con forever](images/Solutions/part1/14-forever-primer-intento.png)

Unos segundos después `forever list` ya muestra la app corriendo y `curl` responde `The answer is 8`:

![App corriendo con forever](images/Solutions/part1/15-forever-app-corriendo.png)

#### 6. Prueba desde el navegador

`http://20.51.178.24:3000/fibonacci/6` responde **`The answer is 8`**.

![Navegador fibonacci 6](images/Solutions/part1/16-navegador-fibonacci-6.png)

#### 7. Tiempos de respuesta (D2s_v4)

Hicimos las 10 peticiones una por una desde el navegador y medimos el tiempo con la pestaña *Network* de las herramientas de desarrollador. Las filas `content_main.js`, `default_config...` y `locales.json` son de las extensiones del navegador, no de la aplicación.

![Tiempos D2s 1](images/Solutions/part1/17-tiempos-d2s-1.png)
![Tiempos D2s 2](images/Solutions/part1/18-tiempos-d2s-2.png)
![Tiempos D2s 3](images/Solutions/part1/19-tiempos-d2s-3.png)

La tabla completa está en la respuesta a la pregunta 4.

#### 8. Consumo de CPU (D2s_v4)

![CPU D2s](images/Solutions/part1/20-cpu-d2s.png)

#### 9. Prueba de carga con Newman (2 usuarios a la vez, D2s_v4)

Cambiamos `VM1` por `20.51.178.24` en `[ARSW_LOAD-BALANCING_AZURE].postman_environment.json` y lanzamos los dos Newman en paralelo desde Git Bash, en nuestro PC (no dentro de la VM):

```
newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e "[ARSW_LOAD-BALANCING_AZURE].postman_environment.json" -n 10 &
newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e "[ARSW_LOAD-BALANCING_AZURE].postman_environment.json" -n 10
```

![Newman 2 hilos D2s ejecución](images/Solutions/part1/21-newman-2hilos-d2s-ejecucion.png)
![Newman 2 hilos D2s resumen](images/Solutions/part1/22-newman-2hilos-d2s-resumen.png)
![CPU Newman 2 hilos D2s](images/Solutions/part1/23-cpu-newman-2hilos-d2s.png)

#### 10. Escalamiento vertical a `Standard_D4s_v4`

En *Disponibilidad y escala → Tamaño* elegimos `D4s_v4`. Azure avisa que la VM se va a reiniciar:

![Confirmación del cambio de tamaño](images/Solutions/part1/24-resize-d4s-confirmacion.png)
![Notificación del cambio de tamaño](images/Solutions/part1/25-resize-d4s-notificacion.png)

Después del reinicio la app ya no estaba corriendo: tuvimos que conectarnos otra vez por SSH y volver a ejecutar `forever start`:

![App después del resize](images/Solutions/part1/26-app-tras-resize.png)

#### 11. Repetición de los pasos 7, 8 y 9 con D4s_v4

Tiempos en el navegador (los `304` significan que el navegador ya tenía esa respuesta guardada; el servidor igual hizo todo el cálculo, por eso el tiempo sí cuenta):

![Tiempos D4s](images/Solutions/part1/27-tiempos-d4s.png)

Newman con 2 usuarios a la vez:

![Newman 2 hilos D4s](images/Solutions/part1/28-newman-2hilos-d4s-resumen.png)

CPU de toda la sesión, de izquierda a derecha: Newman con D2s_v4 (~49 %), un hueco punteado cerca de las 5:45 p. m. que es el reinicio por el cambio de tamaño, las pruebas del navegador con D4s_v4 (~8 %) y Newman con D4s_v4 (~16 %):

![CPU D2s vs D4s](images/Solutions/part1/29-cpu-d2s-vs-d4s.png)

#### 12. Evaluación del escenario de calidad

| Prueba | Peticiones fallidas | ¿Todas respondidas? | CPU máxima | ¿CPU ≤ 70 %? | ¿Cumple? |
|---|---|---|---|---|---|
| D2s_v4 · 2 usuarios | 9 de 20 (45 %) | ❌ | ~49 % | ✅ | ❌ |
| D4s_v4 · 2 usuarios | 8 de 20 (40 %) | ❌ | ~16-22 % | ✅ | ❌ |
| D4s_v4 · 4 usuarios | 16 de 40 (40 %) | ❌ | ~25 % | ✅ | ❌ |

**Conclusión: con escalamiento vertical NO se cumple el escenario de calidad.** La CPU nunca pasa del 70 %, pero en todas las pruebas con varios usuarios se pierde cerca del 40 % de las peticiones. Duplicar la máquina duplicó el costo y casi no cambió nada, porque el problema no es que falte hardware: la aplicación atiende las peticiones de a una (un solo hilo) y no aprovecha los núcleos extra.

#### 13. Volver al tamaño inicial

Al terminar las pruebas devolvimos la VM al tamaño `Standard_D2s_v4` (mismo procedimiento del paso 10), para no seguir pagando el doble. Después eliminamos el grupo `SCALABILITY_LAB` para empezar la Parte 2 en un grupo de recursos limpio.

![Vuelta a D2s_v4](images/Solutions/part1/33-resize-vuelta-d2s.png)

**Preguntas**

**1. ¿Cuántos y cuáles recursos crea Azure junto con la VM?**

Además de la máquina virtual, Azure crea **5 recursos más**, 6 en total (ver imagen del paso 1):

| Recurso | Nombre en nuestro grupo |
|---|---|
| Máquina virtual | `VERTICAL-SCALABILITY` |
| Disco del sistema operativo | `VERTICAL-SCALABILITY_OsDisk_1_...` |
| Interfaz de red (NIC) | `vertical-scalability523` |
| Dirección IP pública | `VERTICAL-SCALABILITY-ip` |
| Grupo de seguridad de red (NSG) | `VERTICAL-SCALABILITY-nsg` |
| Red virtual (VNet), con su subred | `vnet-eastus-1` |

**2. ¿Brevemente describa para qué sirve cada recurso?**

* **Máquina virtual:** es el "computador" que alquilamos en la nube: CPU, memoria y sistema operativo (Ubuntu). Ahí corre la FibonacciApp.
* **Disco:** es el disco duro de la VM. Guarda Ubuntu, Node.js y la aplicación, y no se pierde aunque la VM se apague o se reinicie.
* **Interfaz de red (NIC):** es la "tarjeta de red" de la VM. La conecta a la red virtual y tiene asociadas la IP privada y la IP pública.
* **IP pública:** es la dirección con la que se llega a la VM desde Internet (`20.51.178.24`). Sin ella no podríamos entrar por SSH ni abrir la app en el navegador.
* **Grupo de seguridad de red (NSG):** es el "firewall". Una lista de reglas que dice qué tráfico puede entrar o salir; por ejemplo, SSH por el 22 y la app por el 3000.
* **Red virtual y subred:** es la red privada donde vive la VM dentro de Azure (en nuestro caso le dio la IP privada `172.16.0.4`). Permite que varios recursos se comuniquen entre sí, algo que se usa en la Parte 2.

**3. ¿Al cerrar la conexión ssh con la VM, por qué se cae la aplicación que ejecutamos con el comando `npm FibonacciApp.js`? ¿Por qué debemos crear un *Inbound port rule* antes de acceder al servicio?**

* Cuando ejecutamos la app directamente desde la terminal, el programa queda "amarrado" a esa sesión de SSH. Al cerrarla, Linux cierra también todo lo que se abrió desde ella, y la app se apaga. `forever` la deja corriendo en segundo plano, separada de nuestra sesión, y además la vuelve a levantar si se cae. Por eso, después de cerrar SSH, la app seguía respondiendo.
* Por seguridad, Azure bloquea por defecto todo el tráfico que entra desde Internet; solo dejó abierto el puerto 22 de SSH, que elegimos al crear la VM. La app escucha en el puerto **3000**, así que hasta que no creamos la regla `Port_3000` el navegador no podía llegar a ella, aunque estuviera corriendo.

**4. Adjunte tabla de tiempos e interprete por qué la función tarda tando tiempo.**

| n | D2s_v4 (2 vCPU) | D4s_v4 (4 vCPU) |
|---|---|---|
| 1.000.000 | 8,52 s | 7,80 s |
| 1.010.000 | 8,89 s | 8,03 s |
| 1.020.000 | 8,90 s | 8,35 s |
| 1.030.000 | 9,09 s | 8,55 s |
| 1.040.000 | 9,13 s | 8,75 s |
| 1.050.000 | 9,32 s | 8,90 s |
| 1.060.000 | 9,48 s | 9,12 s |
| 1.070.000 | 9,54 s | 9,24 s |
| 1.080.000 | 9,61 s | 9,35 s |
| 1.090.000 | 9,86 s | 9,12 s |
| **Promedio** | **9,23 s** | **8,72 s** |

Tarda tanto por la forma en que está hecho el cálculo (`FibonacciService.js`):

* Hace un ciclo que suma **un millón de veces** para llegar al resultado.
* Los números de Fibonacci crecen muchísimo: el número 1.000.000 tiene unos **209.000 dígitos** (por eso cada respuesta pesa 209 kB). Cada suma no es instantánea: hay que recorrer todos esos dígitos, como cuando sumamos a mano.
* Entonces, cuanto más grande es n, más sumas hay que hacer **y** cada suma es más larga. Por eso, cuando n crece un 9 % (de 1.000.000 a 1.090.000), el tiempo crece más, un 15,7 % (de 8,52 s a 9,86 s).
* Además, todo ese trabajo lo hace **un solo núcleo**, porque Node.js atiende con un solo hilo.

**5. Adjunte imágen del consumo de CPU de la VM e interprete por qué la función consume esa cantidad de CPU.**

![CPU D2s](images/Solutions/part1/20-cpu-d2s.png)

Con las 10 peticiones del navegador el pico fue de **37,21 %**. La función consume mucha CPU porque se pasa varios segundos haciendo sumas sin parar: no espera ni a la red ni al disco, solo calcula.

¿Por qué no llega al 100 %? Porque Node.js usa **un solo hilo**, es decir, solo puede ocupar **1 de los 2 núcleos** de la VM. Ese núcleo está al máximo, pero el otro casi no hace nada, así que el límite real es del **50 %**. Además Azure promedia por minuto, y entre una petición y otra había unos segundos de pausa mientras escribíamos la siguiente URL, por eso quedó en 37 %. En la prueba con Newman (peticiones sin pausa) sí llegó a **~49 %**, prácticamente el límite.

**6. Adjunte la imagen del resumen de la ejecución de Postman. Interprete:**

![Newman 2 hilos D2s resumen](images/Solutions/part1/22-newman-2hilos-d2s-resumen.png)

| Ejecución (D2s_v4) | Peticiones | Fallidas | Promedio | Mín / Máx |
|---|---|---|---|---|
| Newman A | 10 | 4 (iteraciones 3, 5, 7, 9) | 11,4 s | 9,1 s / 18,1 s |
| Newman B | 10 | 5 (iteraciones 2, 4, 6, 8, 10) | 13,0 s | 9,2 s / 18,4 s |
| **Total** | **20** | **9 (45 %)** | | |

* **Tiempos de ejecución de cada petición:** cuando una petición llega sola tarda lo normal (~9 s). Cuando llegan las dos al mismo tiempo, la app las atiende **una detrás de otra**: la primera tarda ~9 s y la segunda tiene que esperar a que termine la primera, por eso tarda ~18 s, el doble. Es como una caja de supermercado con una sola cajera.
* **Si hubo fallos documentelos y explique:** sí, **9 de 20 peticiones fallaron** con el error `read ECONNRESET`, que significa que el servidor "colgó" la conexión sin responder. Mientras la app está ocupada calculando la petición de un usuario no puede atender a nadie más. La conexión del otro usuario se queda esperando, y como pasan varios segundos sin actividad, Node la cierra justo cuando llega la siguiente petición. Por eso los fallos van alternados: una petición sí y la siguiente no.

**7. ¿Cuál es la diferencia entre los tamaños `B2ms` y `B1ls` (no solo busque especificaciones de infraestructura)?**

| | `B1ls` | `B2ms` |
|---|---|---|
| vCPU / RAM | 1 / 0,5 GiB | 2 / 8 GiB |
| Rendimiento base de CPU | 5 % de su vCPU | 60 % del total |
| Costo aprox. | 0,0052 US$/h | ~0,083 US$/h |

* Las dos son de la **serie B ("burstable")**: están pensadas para cargas que pasan casi todo el tiempo tranquilas y de vez en cuando tienen picos. Cuando la VM está por debajo de su rendimiento base **acumula créditos**, y en los picos los gasta para usar el 100 % del procesador. Si se le acaban los créditos, Azure la "frena" hasta su rendimiento base.
* La gran diferencia es cuánto aguantan: la `B1ls` solo puede usar el 5 % de un núcleo de forma sostenida y tiene tan poca memoria (0,5 GiB) que con un cálculo largo como el nuestro se quedaría sin créditos y sin memoria rápidamente. La `B2ms` tiene 16 veces más memoria y puede trabajar sostenidamente al 60 %, por eso aguanta mucho mejor las pruebas de carga, pero cuesta unas 16 veces más.
* La `B1ls` además solo se ofrece para Linux y está pensada para cosas muy pequeñas (servidores de prueba, tareas livianas).

En nuestro caso, como la serie B no estaba disponible, comparamos **`D2s_v4` vs `D4s_v4`**. Estas **no** son burstable: siempre entregan el 100 % de su CPU, sin créditos. La `D4s_v4` tiene el doble de todo (4 vCPU, 16 GiB, 8 discos de datos, 6400 IOPS) y cuesta el doble (0,192 vs 0,096 US$/h), pero sus núcleos son igual de rápidos que los de la `D2s_v4`.

**8. ¿Aumentar el tamaño de la VM es una buena solución en este escenario?, ¿Qué pasa con la FibonacciApp cuando cambiamos el tamaño de la VM?**

**No es una buena solución.** Pagamos el doble y las fallas casi no cambiaron (45 % → 40 %). La app usa un solo hilo, así que los núcleos nuevos se quedan sin usar: es como contratar más cajeras pero dejar abierta una sola caja.

Al cambiar el tamaño, Azure **reinicia la VM**, así que la FibonacciApp **se cae** y deja de responder durante unos minutos. Además no vuelve sola: tuvimos que entrar otra vez por SSH y ejecutar `forever start FibonacciApp.js` (paso 10).

**9. ¿Qué pasa con la infraestructura cuando cambia el tamaño de la VM? ¿Qué efectos negativos implica?**

Azure apaga la VM, la pasa a un servidor físico que tenga el nuevo tamaño y la vuelve a encender. El disco (con Ubuntu, Node y la app) se conserva, y la IP pública también, porque es estática; por eso la dirección siguió siendo `20.51.178.24`.

Efectos negativos:
* **El servicio se cae** mientras dura el cambio. En la gráfica de CPU se ve como un hueco punteado cerca de las 5:45 p. m.: durante esos minutos no había VM ni datos.
* **Hay que intervenir a mano** para volver a levantar la aplicación.
* **El costo sube** (en nuestro caso al doble) aunque el rendimiento casi no mejore.
* **Hay un techo**: no se puede crecer para siempre, y no todos los tamaños están disponibles (como nos pasó con la serie B).
* **Sigue siendo una sola máquina**: si falla, se cae todo el sistema.

**10. ¿Hubo mejora en el consumo de CPU o en los tiempos de respuesta? Si/No ¿Por qué?**

**Prácticamente no.**

| | D2s_v4 | D4s_v4 | Cambio |
|---|---|---|---|
| Tiempo promedio (1 usuario) | 9,23 s | 8,72 s | −5,5 % |
| Peticiones fallidas (2 usuarios) | 45 % | 40 % | casi igual |
| CPU máxima con Newman | ~49 % | ~16-22 % | baja, pero engañosa |

* Los tiempos mejoraron apenas un 5 %, que puede ser solo variación normal. Los núcleos de las dos VM son de la misma generación, y como la app usa uno solo, cada petición tarda casi lo mismo.
* El porcentaje de CPU bajó, pero no porque la app trabaje mejor: ahora hay 4 núcleos en vez de 2, y la app sigue ocupando solo uno. Ese único núcleo sigue igual de ocupado; simplemente "pesa" menos en el promedio (su límite pasó del 50 % al 25 %).
* Las fallas siguen en ~40 %, porque el problema de fondo (un solo hilo atendiendo a todos) no cambió.

**11. Aumente la cantidad de ejecuciones paralelas del comando de postman a `4`. ¿El comportamiento del sistema es porcentualmente mejor?**

![Newman 4 hilos D4s 1](images/Solutions/part1/30-newman-4hilos-d4s-1.png)
![Newman 4 hilos D4s 2](images/Solutions/part1/31-newman-4hilos-d4s-2.png)
![CPU Newman 4 hilos D4s](images/Solutions/part1/32-cpu-newman-4hilos-d4s.png)

| Ejecución (D4s_v4) | Peticiones | Fallidas | Promedio | Mín / Máx |
|---|---|---|---|---|
| 1 | 10 | 4 | 22,7 s | 15,3 s / 30,7 s |
| 2 | 10 | 4 | 26,0 s | 15,3 s / 30,6 s |
| 3 | 10 | 4 | 25,6 s | 7,9 s / 30,7 s |
| 4 | 10 | 4 | 28,0 s | 15,3 s / 37,2 s |
| **Total** | **40** | **16 (40 %)** | **~25,6 s** | |

**No, no es porcentualmente mejor:**
* El porcentaje de fallas se queda igual: **40 %**, como con 2 usuarios.
* Los usuarios esperan **más del doble**: de ~11 s pasamos a ~25,6 s en promedio, porque ahora hay 4 peticiones haciendo fila detrás del único hilo de la app.
* La CPU se quedó plana en **~25 %**: un núcleo trabajando al máximo y los otros tres sin hacer nada.
* La app termina despachando siempre más o menos **una respuesta cada 8 segundos**, sin importar si hay 2 o 4 usuarios. Los usuarios de más no se atienden más rápido; solo esperan más.

### Parte 2 - Escalabilidad horizontal

En vez de agrandar una sola máquina, en esta parte ponemos **varias máquinas iguales** detrás de un **balanceador de carga** que reparte a los usuarios entre ellas. Antes de empezar borramos el grupo de recursos de la Parte 1 y trabajamos en un grupo `SCALABILITY_LAB` nuevo, en la región **East US**.

> **Ajustes respecto al enunciado.** La suscripción *Azure for Students* nos puso tres límites que cambiaron un poco el montaje (todo está documentado con capturas en la sección de las VM):
>
> | Límite de la suscripción | Qué hicimos |
> |---|---|
> | La serie B no está disponible (igual que en la Parte 1) | Los nodos son `Standard_D2s_v4` (2 vCPU, 8 GiB) |
> | Máximo **4 vCPU** de la familia DSv4 y **6 vCPU** en total en la región | VM1 y VM2 son `D2s_v4`; **VM3 es `Standard_D2as_v4`** (otra familia, también 2 vCPU y 8 GiB, mismo precio). **VM4 no se pudo crear**: no quedaban vCPU y Azure rechazó el aumento de cuota |
> | Máximo **3 IP públicas** | Las usan el balanceador, VM1 y VM2. **VM3 se creó sin IP pública**: le llega el tráfico por el balanceador y la administramos entrando por VM1 |
>
> El orden también cambió un poco: en el portal actual, al crear el *Backend Pool* hay que elegir la red virtual, así que la creamos antes (el paso 5 se hizo justo después del 1). Aquí lo presentamos en el orden del enunciado.

#### Crear el Balanceador de Carga

**1. Balanceador de carga.** `SCALABILITY_LAB_LOAD_BALANCER`, SKU **Estándar**, tipo **Público**, nivel **Regional**:

![LB datos básicos](images/Solutions/parte2/01-lb-datos-basicos.png)

Con una IP de front-end nueva, `SCALABILITY_LAB_LOAD_BALANCER_IP` (IP pública Standard):

![LB IP de front-end](images/Solutions/parte2/02-lb-ip-frontend.png)
![LB creado](images/Solutions/parte2/03-lb-creado.png)

La IP pública del balanceador quedó en **`20.246.233.87`**. Todavía sin grupo de back-end, regla ni sondeo:

![LB información general](images/Solutions/parte2/04-lb-informacion-general.png)

**2. Backend Pool.** `SCALABILITY_LAB_BACKEND_POOL`, asociado a la red virtual y configurado por **NIC** (tarjeta de red). Se crea vacío; cada VM se agrega sola cuando la creamos:

![Backend pool](images/Solutions/parte2/05-backend-pool-crear.png)
![Backend pool creado](images/Solutions/parte2/06-backend-pool-creado.png)

**3. Health Probe.** `SCALABILITY_LAB_HEALTH_PROBE`: protocolo **TCP**, puerto **3000**, cada **5 segundos**:

![Health probe](images/Solutions/parte2/07-health-probe-crear.png)
![Health probe creado](images/Solutions/parte2/08-health-probe-creado.png)

**4. Load Balancing Rule.** `SCALABILITY_LAB_LOAD_BALANCING_RULE`: lo que llega al front-end por el puerto **80** se reparte entre el grupo de back-end, al puerto **3000** de cada VM. Usa el sondeo anterior y **sin persistencia de sesión**:

![Regla de balanceo](images/Solutions/parte2/09-regla-balanceo.png)

**5. Virtual Network.** `SCALABILITY_LAB_VIRTUAL_NETWORK` con espacio de direcciones **`10.1.0.0/16`** y la subred `SCALABILITY_LAB_SUBNET` **`10.1.0.0/24`**. Dejamos desmarcada la "subred privada", para que las VM pudieran salir a Internet a descargar Node y la app:

![VNet y subred](images/Solutions/parte2/10-vnet-direcciones-subred.png)
![VNet creada](images/Solutions/parte2/11-vnet-creada.png)

#### Crear las maquinas virtuales (Nodos)

Antes de crear las VM revisamos las cuotas de East US. Muchas familias están bloqueadas (**0 de 0**) y la familia DSv4 solo permite **4 vCPU**, es decir, dos VM de 2 vCPU:

![Cuotas bloqueadas](images/Solutions/parte2/12-cuotas-familias-bloqueadas.png)
![Cuota DSv4](images/Solutions/parte2/13-cuota-dsv4-inicial.png)

**1. Configuración básica (VM1).** `VM1`, East US, **zona de disponibilidad 1**, Ubuntu Server 24.04 LTS, `Standard_D2s_v4`, sin Azure Spot y con llave SSH:

![VM1 datos básicos](images/Solutions/parte2/14-vm1-datos-basicos.png)

**2. Networking.** Red `SCALABILITY_LAB_VIRTUAL_NETWORK`, subred `SCALABILITY_LAB_SUBNET`, IP pública nueva `VM1-ip`. En la sección *Equilibrio de carga* elegimos **Azure Load Balancer → `SCALABILITY_LAB_LOAD_BALANCER` → `SCALABILITY_LAB_BACKEND_POOL`**:

![VM1 redes](images/Solutions/parte2/15-vm1-redes.png)

**3. Network Security Group.** En modo **Avanzado** creamos `LOAD_BALANCER_NSG` con la regla de SSH y una regla de entrada **TCP 3000, Permitir, prioridad 1010**. Por un error de tipeo la regla quedó con el nombre `Port_3030`, pero el puerto sí es el 3000. VM2 y VM3 reutilizan este mismo NSG:

![Regla 3000](images/Solutions/parte2/16-vm1-nsg-regla-3000.png)
![NSG creado](images/Solutions/parte2/17-nsg-load-balancer-nsg.png)

**4. VM1 creada.** Zona 1, IP privada `10.1.0.4`, IP pública `40.90.225.78`. El grupo de back-end ya aparece usado por la regla de balanceo:

![VM1 implementación](images/Solutions/parte2/18-vm1-implementacion.png)
![VM1 información general](images/Solutions/parte2/19-vm1-informacion-general.png)
![Backend pool usado por la regla](images/Solutions/parte2/20-backend-pool-usado-por-regla.png)

**VM2.** Igual que VM1, pero en la **zona 2**: IP privada `10.1.0.5`, IP pública `52.249.220.114`, con el NSG existente:

![VM2 datos básicos](images/Solutions/parte2/21-vm2-datos-basicos.png)
![VM2 redes](images/Solutions/parte2/22-vm2-redes.png)
![VM2 implementación](images/Solutions/parte2/23-vm2-implementacion.png)

**Límite de cuotas.** Con VM1 y VM2 la familia DSv4 quedó llena (**4 de 4**) y la región en **4 de 6** vCPU. Pedimos subir las dos cuotas a 8 y Azure **rechazó la solicitud**:

![Cuotas llenas](images/Solutions/parte2/24-cuotas-llenas.png)
![Solicitud de cuota](images/Solutions/parte2/25-solicitud-cuota.png)
![Cuota rechazada](images/Solutions/parte2/26-cuota-rechazada.png)

**VM3.** Con los 2 vCPU que quedaban libres la creamos en la **zona 3** con `Standard_D2as_v4` (familia DASv4: AMD, 2 vCPU, 8 GiB, mismo precio):

![VM3 datos básicos](images/Solutions/parte2/27-vm3-datos-basicos.png)

El primer intento, con IP pública (`vm3-ip`), falló porque la suscripción solo admite **3 IP públicas** (`ResourceCountExceedsLimitDueToTemplate`):

![VM3 primer intento](images/Solutions/parte2/28-vm3-redes-primer-intento.png)

Así que la creamos **sin IP pública**. Solo tiene IP privada (`10.1.0.6`):

![VM3 sin IP pública](images/Solutions/parte2/29-vm3-implementacion-sin-ip.png)

Una VM sin IP pública detrás de un balanceador Standard no puede salir a Internet. Para que VM3 pudiera descargar Node y la app, cambiamos la opción **SNAT de salida** de la regla para que use el puerto predeterminado del balanceador. En producción lo recomendado sería una regla de salida o un NAT Gateway:

![SNAT de salida](images/Solutions/parte2/30-lb-snat-salida.png)

**5. Instalación de la aplicación.** En cada VM ejecutamos los mismos comandos del enunciado, con dos cambios: clonamos nuestro repositorio y usamos `source ~/.bashrc`, porque el enunciado tiene la ruta `/home/vm1` fija:

```
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
source ~/.bashrc
nvm install --lts

git clone https://github.com/JuanTorres1303/ARSWLAB07.git
cd ARSWLAB07/FibonacciApp
npm install
npm install -g forever
forever start FibonacciApp.js
```

VM1 (`ssh -i ~/.ssh/arsw_lab07 vm1@40.90.225.78`):

![VM1 SSH](images/Solutions/parte2/31-vm1-ssh.png)
![VM1 app con forever](images/Solutions/parte2/32-vm1-app-forever.png)

VM2 (al crearla, el usuario quedó en mayúsculas: `VM2@52.249.220.114`):

![VM2 SSH](images/Solutions/parte2/33-vm2-ssh.png)
![VM2 app con forever](images/Solutions/parte2/34-vm2-app-forever.png)

VM3 no tiene IP pública, así que entramos **a través de VM1**, que hace de puente (*jump host*). En `~/.ssh/config`:

```
Host vm1-jump
    HostName 40.90.225.78
    User vm1
    IdentityFile ~/.ssh/arsw_lab07

Host vm3
    HostName 10.1.0.6
    User VM3
    IdentityFile ~/.ssh/arsw_lab07
    ProxyJump vm1-jump
```

![VM3 SSH por VM1](images/Solutions/parte2/35-vm3-ssh-por-vm1.png)
![VM3 app con forever](images/Solutions/parte2/36-vm3-app-forever.png)

#### Probar el resultado final de nuestra infraestructura

**1. Servicios básicos por la IP del balanceador** (puerto 80, sin escribir `:3000`). Con solo VM1 lista ya respondía:

![LB fibonacci 6](images/Solutions/parte2/37-lb-fibonacci-6.png)

Con los tres nodos:

```
http://20.246.233.87/
http://20.246.233.87/fibonacci/1
```

![LB Hello World](images/Solutions/parte2/38-lb-hello-world.png)
![LB fibonacci 1](images/Solutions/parte2/39-lb-fibonacci-1.png)

Las tres VM dentro del grupo de back-end, cada una en una zona distinta:

![Backend pool con 3 VM](images/Solutions/parte2/40-backend-pool-3-vms.png)

> **El sondeo de estado funcionando de verdad.** En la primera ronda de pruebas, la app no había quedado corriendo en VM3 (los comandos de `forever` se pegaron por error en la terminal de VM2). El sondeo detectó que VM3 no respondía en el puerto 3000 y el balanceador **dejó de mandarle tráfico**: la CPU de VM3 quedó en 0 % y todo lo atendieron VM1 y VM2. Lo comprobamos desde VM1 con `curl http://10.1.0.6:3000/`, que no respondía. Después arrancamos `forever` en VM3 y repetimos las pruebas. Los resultados con 3 nodos sanos son los que se muestran abajo; los de esa primera ronda (2 nodos sanos) aparecen solo como dato en las tablas.

**2. Pruebas de carga con `newman` (2 usuarios a la vez).** Usamos la colección de `FibonacciApp/postman/part2`, que llama a `http://{{loadbalancer}}/fibonacci/1000000`, con `loadbalancer = 20.246.233.87`:

```
newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e "[ARSW_LOAD-BALANCING_AZURE].postman_environment.json" -n 10 &
newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e "[ARSW_LOAD-BALANCING_AZURE].postman_environment.json" -n 10
```

![Newman 2 hilos 3 nodos](images/Solutions/parte2/41-newman-2hilos-3nodos.png)

| Ejecución | Peticiones | Fallidas | Promedio | Mín / Máx | Desviación |
|---|---|---|---|---|---|
| A | 10 | **0** | 6,8 s | 6,7 / 6,9 s | 59 ms |
| B | 10 | **0** | 8,3 s | 8,2 / 8,4 s | 65 ms |

El balanceador mandó a cada usuario a un nodo distinto (VM1 y VM3). Cada nodo atendió a un solo usuario, así que nadie hizo fila. La ejecución A tardó un poco menos porque cayó en VM3, cuyo procesador AMD (`D2as_v4`) calcula más rápido que el Intel de las `D2s_v4`:

![CPU 2 hilos 3 nodos](images/Solutions/parte2/42-cpu-2hilos-3nodos.png)

El informe comparativo con la Parte 1 (tiempos, peticiones exitosas y costos) está en la respuesta **"Informe de newman 1 (Punto 2)"**, más abajo.

**3. Cuarta máquina virtual y 4 usuarios en paralelo.** Como explicamos arriba, **la VM4 no se pudo crear**: la región ya estaba en el límite de vCPU, la suscripción no admite una cuarta IP pública y Azure rechazó el aumento de cuota. Por eso hicimos la prueba de **4 usuarios contra los 3 nodos**:

```
newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e "[ARSW_LOAD-BALANCING_AZURE].postman_environment.json" -n 10 &
newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e "[ARSW_LOAD-BALANCING_AZURE].postman_environment.json" -n 10 &
newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e "[ARSW_LOAD-BALANCING_AZURE].postman_environment.json" -n 10 &
newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e "[ARSW_LOAD-BALANCING_AZURE].postman_environment.json" -n 10
```

![Newman 4 hilos 3 nodos 1](images/Solutions/parte2/43-newman-4hilos-3nodos-1.png)
![Newman 4 hilos 3 nodos 2](images/Solutions/parte2/44-newman-4hilos-3nodos-2.png)

| Ejecución | Peticiones | Fallidas | Promedio | Mín / Máx |
|---|---|---|---|---|
| 1 | 10 | 2 | 9,2 s | 8,0 / 14,0 s |
| 2 | 10 | 1 | 10,0 s | 7,3 / 23,7 s |
| 3 | 10 | 2 | 12,5 s | 8,0 / 23,1 s |
| 4 | 10 | 2 | 11,9 s | 8,1 / 30,5 s |
| **Total** | **40** | **7 (17,5 %)** | **~10,9 s** | |

**Comportamiento de la CPU de los nodos.** En la gráfica se ven las tres VM. VM2 llega a ~49 %, que es el máximo de un solo hilo en una máquina de 2 vCPU, y VM1 y VM3 también suben. Las líneas punteadas del final son los últimos minutos, que Azure todavía no había terminado de procesar cuando tomamos la captura:

![CPU 4 hilos 3 nodos](images/Solutions/parte2/45-cpu-4hilos-3nodos.png)

**¿Por qué aumentó la tasa de éxito con este estilo de escalabilidad?**

| 4 usuarios a la vez | Fallidas | Promedio | Respuestas correctas por minuto |
|---|---|---|---|
| Vertical: 1 VM `D4s_v4` (4 vCPU) | 16 de 40 (**40 %**) | 25,6 s | ~7,8 |
| Horizontal: LB con 2 nodos sanos | 14 de 40 (35 %) | 12,8 s | ~13,8 |
| **Horizontal: LB con 3 nodos** | **7 de 40 (17,5 %)** | **10,9 s** | **~18,5** |

* La FibonacciApp atiende **una petición a la vez** (Node.js usa un solo hilo). En la Parte 1, agrandar la máquina no servía: era como contratar más cajeras y dejar abierta **una sola caja**.
* Con escalamiento horizontal, cada VM es **otra caja abierta**, con su propio hilo. El balanceador reparte a los usuarios entre las cajas, así que hay menos gente haciendo fila en cada una. Menos fila significa menos conexiones que se cansan de esperar (`ECONNRESET`) y respuestas más rápidas.
* Por eso, cada nodo que agregamos mejoró el resultado: con 2 nodos sanos falló el 35 % y con 3 nodos el 17,5 %. Además, el sistema despacha más del doble de respuestas por minuto que la VM grande de la Parte 1.
* Todavía falla algo porque hay **4 usuarios y solo 3 nodos**: en algún momento dos usuarios caen en la misma VM y vuelve a pasar lo de la Parte 1 en ese nodo. Con la VM4 los fallos bajarían más, aunque el reparto por *hash* no garantiza exactamente un usuario por nodo.

**Evaluación del escenario de calidad**

| Prueba | Fallidas | ¿Todas respondidas? | CPU máx. por nodo | ¿≤ 70 %? | ¿Cumple? |
|---|---|---|---|---|---|
| LB 3 nodos · 2 usuarios | 0 de 20 | ✅ | ~41 % | ✅ | ✅ |
| LB 3 nodos · 4 usuarios | 7 de 40 | ❌ | ~49 % | ✅ | ❌ |

Con escalamiento horizontal **sí se cumple el escenario cuando hay al menos un nodo por cada usuario concurrente**: con 2 usuarios todas las peticiones respondieron y ningún nodo pasó del 50 %. Con 4 usuarios y 3 nodos todavía se pierden peticiones; para cumplirlo haría falta otro nodo (que la cuota no permitió) o cambiar la aplicación para que no bloquee su único hilo.

**Preguntas**

**¿Cuáles son los tipos de balanceadores de carga en Azure y en qué se diferencian?, ¿Qué es SKU, qué tipos hay y en qué se diferencian?, ¿Por qué el balanceador de carga necesita una IP pública?**

Tipos de balanceadores en Azure:

| Servicio | Nivel | Alcance | Para qué sirve |
|---|---|---|---|
| **Azure Load Balancer** (el que usamos) | Capa 4 (TCP/UDP) | Regional (también hay un nivel global) | Reparte conexiones sin mirar su contenido. Muy rápido y sirve para cualquier protocolo |
| **Application Gateway** | Capa 7 (HTTP/HTTPS) | Regional | Entiende la petición web: puede enrutar por URL, manejar certificados SSL, usar cookies de sesión y tener firewall de aplicaciones (WAF) |
| **Azure Front Door** | Capa 7 (HTTP/HTTPS) | Global | Puerta de entrada mundial con caché (CDN) y WAF; manda a cada usuario a la región más cercana |
| **Traffic Manager** | DNS | Global | Responde con la dirección de la región más conveniente; no pasa el tráfico por él |
| **Gateway Load Balancer** | Capa 3 | Regional | Inserta equipos de red virtuales (firewalls, inspección) en el camino del tráfico |

**SKU** (*Stock Keeping Unit*) es la "versión" o nivel de producto de un recurso. Define qué funciones tiene, qué garantías da y cuánto cuesta. En el Load Balancer hay tres:

| | Basic | **Standard** (el que usamos) | Gateway |
|---|---|---|---|
| Estado | Retirado el 30 de septiembre de 2025 | Vigente | Vigente |
| Garantía (SLA) | Ninguna | 99,99 % | 99,99 % |
| Zonas de disponibilidad | No | Sí (redundancia de zona) | Sí |
| Tamaño del back-end | Hasta 300 instancias | Hasta 1000 instancias | — |
| Seguridad | Abierto por defecto | **Cerrado por defecto**: hay que permitir el tráfico con un NSG | — |

Las IP públicas también tienen SKU: *Basic* (también retirada en septiembre de 2025) y *Standard*, que es estática, segura por defecto y admite zonas. Un Load Balancer Standard solo acepta IP públicas Standard.

**¿Por qué necesita una IP pública?** Porque es la **única puerta de entrada** que conocen los usuarios. Las VM viven en una red privada (`10.1.0.x`), que no es alcanzable desde Internet. La IP pública `20.246.233.87` le da al sistema una dirección fija, sin importar cuántas VM haya detrás o cuáles estén encendidas. Incluso VM3, que no tiene IP pública propia, recibe usuarios gracias a ella. (Un balanceador *interno* no necesita IP pública, pero solo sirve dentro de la red privada).

**¿Cuál es el propósito del *Backend Pool*?**

Es la **lista de máquinas que atienden** el tráfico. El balanceador solo reparte entre los miembros de ese grupo. En nuestro caso, `SCALABILITY_LAB_BACKEND_POOL` tiene las tarjetas de red de VM1, VM2 y VM3, cada una en una zona distinta (ver imagen del back-end pool con 3 VM). Para escalar horizontalmente basta con agregar otra VM al grupo, sin cambiar la IP que usan los usuarios.

**¿Cuál es el propósito del *Health Probe*?**

Es un "¿sigues vivo?" que el balanceador le pregunta a cada máquina del grupo cada pocos segundos. Nuestro sondeo intenta conectarse al **puerto 3000 por TCP cada 5 segundos**. Si una VM no contesta, el balanceador **deja de enviarle usuarios** hasta que vuelva a responder, y así nadie es enviado a una máquina caída.

Lo vimos en la práctica: cuando la app no estaba corriendo en VM3, el sondeo la marcó como no disponible y todo el tráfico se fue a VM1 y VM2. La CPU de VM3 se quedó en 0 % y ningún usuario recibió errores por culpa de esa máquina.

**¿Cuál es el propósito de la *Load Balancing Rule*? ¿Qué tipos de sesión persistente existen, por qué esto es importante y cómo puede afectar la escalabilidad del sistema?**

La regla une todas las piezas. Dice: "lo que llegue a **esta IP** (front-end) por **este puerto** (80) y protocolo (TCP), repártelo entre **este grupo** (back-end pool), al **puerto 3000**, solo entre las máquinas que el **sondeo** marque como sanas". Gracias a ella los usuarios entran por `http://20.246.233.87/` sin escribir `:3000`.

Tipos de persistencia de sesión (el portal los llama así):

| Opción | Cómo decide a qué VM mandar | Efecto |
|---|---|---|
| **Ninguna** (la que usamos) | Hash de 5 datos: IP y puerto de origen, IP y puerto de destino y protocolo | Cada conexión nueva puede ir a cualquier VM: el reparto es más parejo |
| **IP de cliente** | Hash de IP de origen + IP de destino | El mismo usuario siempre cae en la misma VM |
| **IP de cliente y protocolo** | Hash de IP de origen + IP de destino + protocolo | Igual que la anterior, separando por protocolo |

**¿Por qué es importante?** Si una aplicación guarda datos del usuario **en la memoria de la VM** (por ejemplo, un carrito de compras o una sesión iniciada), y la siguiente petición cae en otra VM, esos datos "desaparecen". La persistencia evita eso.

**¿Cómo afecta la escalabilidad?** Amarrar a cada usuario a una VM hace que el reparto sea **desigual**: una VM puede quedar llena mientras otras están libres (y si muchos usuarios salen a Internet por la misma IP, por ejemplo desde una universidad, todos caen en la misma VM). Además, al agregar una VM nueva, los usuarios que ya están amarrados no se mueven a ella. Lo ideal es que la aplicación **no guarde estado** en la VM (como nuestra FibonacciApp, que solo calcula) o que lo guarde fuera (una base de datos o una caché compartida como Redis), y dejar la persistencia en **Ninguna**.

**¿Qué es una *Virtual Network*? ¿Qué es una *Subnet*? ¿Para qué sirven los *address space* y *address range*?**

* **Virtual Network (VNet):** es una **red privada propia dentro de Azure**, aislada de las de otros clientes. Ahí las VM se comunican entre sí con direcciones privadas. La nuestra es `SCALABILITY_LAB_VIRTUAL_NETWORK`.
* **Subnet (subred):** es un **pedazo de la VNet**. Sirve para separar grupos de recursos (por ejemplo, servidores web y bases de datos) y aplicarles reglas distintas. Nuestras tres VM están en `SCALABILITY_LAB_SUBNET`.
* **Address space:** es el **rango total de direcciones** que puede usar la red: `10.1.0.0/16`, es decir de `10.1.0.0` a `10.1.255.255` (**65.536 direcciones**).
* **Address range (de la subred):** es la **porción** de ese espacio que le toca a cada subred: `10.1.0.0/24`, de `10.1.0.0` a `10.1.0.255` (**256 direcciones**, de las cuales Azure reserva 5, así que quedan 251 para máquinas). De ahí salieron las IP de nuestras VM: `10.1.0.4`, `10.1.0.5` y `10.1.0.6`. Las subredes de una misma VNet no pueden superponerse, y queda espacio libre para crear más subredes después.

**¿Qué son las *Availability Zone* y por qué seleccionamos 3 diferentes zonas?. ¿Qué significa que una IP sea *zone-redundant*?**

* Una **zona de disponibilidad** es uno o varios **centros de datos físicamente separados** dentro de la misma región (East US), cada uno con su propia energía, refrigeración y red. Si a uno le pasa algo (un corte de luz, un incendio, una falla de red), los otros siguen funcionando.
* Pusimos VM1 en la zona 1, VM2 en la 2 y VM3 en la 3 para que **una falla en un centro de datos no tumbe todo el sistema**: si se cae una zona, el sondeo marca esa VM como no disponible y el balanceador sigue repartiendo entre las otras dos. Con VM en dos o más zonas, Azure garantiza un 99,99 % de disponibilidad, frente al 99,9 % de una sola VM.
* Una IP **zone-redundant** (con redundancia de zona) **no vive en una sola zona**: está replicada en las tres, así que sigue respondiendo aunque una zona se caiga. Es justo lo que necesita la IP de un balanceador, porque si ella falla da igual que las VM estén sanas. En el portal actual no apareció el selector de zona al crear la IP; las IP públicas Standard se crean con redundancia de zona por defecto en las regiones que tienen zonas.

**¿Cuál es el propósito del *Network Security Group*?**

Es el **firewall** de las máquinas: una lista de reglas que dice qué tráfico puede entrar y salir, según origen, destino, puerto y protocolo. Las reglas se revisan por prioridad, de la más baja a la más alta. Nuestro `LOAD_BALANCER_NSG` lo comparten las tres VM y tiene:

| Prioridad | Regla | Para qué |
|---|---|---|
| 1000 | SSH (TCP 22) | Administrar las VM |
| 1010 | TCP 3000 | Que el balanceador y los usuarios lleguen a la app |
| 65000 | AllowVnetInBound | Que las VM de la red se hablen entre sí (así VM1 hace de puente hacia VM3) |
| 65001 | AllowAzureLoadBalancerInBound | Que el sondeo del balanceador pueda revisar las VM |
| 65500 | DenyAllInBound | Todo lo demás se bloquea |

Es especialmente importante con un balanceador **Standard**, porque es "cerrado por defecto": sin la regla del puerto 3000, el balanceador recibiría a los usuarios pero no podría pasárselos a las VM.

**Informe de newman 1 (Punto 2)**

Comparamos la misma prueba (2 usuarios a la vez, 10 peticiones de `fibonacci/1000000` cada uno) en las dos arquitecturas.

*Tiempos de respuesta y peticiones respondidas con éxito:*

| Arquitectura | Exitosas | Fallidas | Promedio | Peor tiempo | Duración | Exitosas por minuto |
|---|---|---|---|---|---|---|
| Vertical: 1 VM `D2s_v4` (2 vCPU) | 11 de 20 | 9 (**45 %**) | ~12,2 s | 18,4 s | 1 min 41 s | ~6,5 |
| Vertical: 1 VM `D4s_v4` (4 vCPU) | 12 de 20 | 8 (**40 %**) | ~10,8 s | 15,8 s | 1 min 35 s | ~7,5 |
| Horizontal: LB + 2 nodos sanos | 20 de 20 | **0** | ~8,3 s | 8,7 s | 1 min 25 s | ~14 |
| **Horizontal: LB + 3 nodos** | **20 de 20** | **0** | **~7,6 s** | **8,4 s** | **1 min 24 s** | **~14,2** |

*Costos aproximados* (precios de lista en East US, Linux, pago por uso; sin contar los discos, unos 2,4 US$/mes por VM):

| Arquitectura | Recursos | Costo por hora | Costo al mes (730 h) |
|---|---|---|---|
| Vertical `D2s_v4` | 1 VM (0,096) + 1 IP pública (0,005) | ~0,10 US$ | ~74 US$ |
| Vertical `D4s_v4` | 1 VM (0,192) + 1 IP pública (0,005) | ~0,20 US$ | ~144 US$ |
| **Horizontal 3 nodos** | 3 VM (3 × 0,096) + Load Balancer Standard (~0,025) + 3 IP públicas (3 × 0,005) | **~0,33 US$** | **~239 US$** |

*Conclusiones:*
* **Peticiones exitosas:** el escalamiento horizontal respondió **todas** las peticiones; el vertical perdió entre el 40 % y el 45 %, aunque le duplicamos el tamaño.
* **Tiempos:** en horizontal cada usuario tuvo "su" máquina, así que todas las respuestas tardaron lo mismo que una petición sola (~7–8 s, con apenas 60–140 ms de variación). En vertical las peticiones hacían fila y algunas tardaban el doble.
* **Costos:** por hora, la arquitectura horizontal cuesta más (~0,33 US$ frente a ~0,20 US$ de la VM grande), porque son 3 máquinas más el balanceador. Pero **rinde mucho más por cada dólar**: con 4 usuarios, la VM grande despachó ~7,8 respuestas correctas por minuto y la horizontal ~18,5. Eso da unos **0,42 US$ por cada 1000 respuestas en vertical, frente a ~0,30 US$ en horizontal**. Además, en horizontal se pueden apagar o quitar nodos cuando hay poca demanda (o automatizarlo con un *VM Scale Set*), mientras que en vertical se paga siempre la máquina grande y cambiarla de tamaño obliga a reiniciarla.

**Presente el Diagrama de Despliegue de la solución.**

```mermaid
flowchart TB
    users["Usuarios / Newman<br/>(PC del estudiante)"]
    admin["Administrador<br/>(SSH con llave arsw_lab07)"]

    subgraph azure["Microsoft Azure · Suscripción Azure for Students · Región East US"]
        subgraph rg["Grupo de recursos SCALABILITY_LAB"]
            lbip["IP pública Standard<br/>SCALABILITY_LAB_LOAD_BALANCER_IP<br/>20.246.233.87"]

            subgraph lb["Azure Load Balancer Standard · SCALABILITY_LAB_LOAD_BALANCER"]
                fe["Front-end<br/>SCALABILITY_LAB_FRONTEND"]
                rule["Regla SCALABILITY_LAB_LOAD_BALANCING_RULE<br/>TCP 80 → 3000 · persistencia: Ninguna"]
                probe["Sondeo SCALABILITY_LAB_HEALTH_PROBE<br/>TCP 3000 cada 5 s"]
                pool["Back-end pool<br/>SCALABILITY_LAB_BACKEND_POOL"]
            end

            nsg["NSG LOAD_BALANCER_NSG<br/>permite TCP 22 y TCP 3000"]

            subgraph vnet["VNet SCALABILITY_LAB_VIRTUAL_NETWORK · 10.1.0.0/16"]
                subgraph subnet["Subred SCALABILITY_LAB_SUBNET · 10.1.0.0/24"]
                    subgraph z1["Zona 1"]
                        vm1["VM1 · Standard_D2s_v4<br/>IP privada 10.1.0.4<br/>IP pública 40.90.225.78<br/>Ubuntu 24.04 · Node 24 · forever<br/>FibonacciApp :3000"]
                    end
                    subgraph z2["Zona 2"]
                        vm2["VM2 · Standard_D2s_v4<br/>IP privada 10.1.0.5<br/>IP pública 52.249.220.114<br/>Ubuntu 24.04 · Node 24 · forever<br/>FibonacciApp :3000"]
                    end
                    subgraph z3["Zona 3"]
                        vm3["vm3 · Standard_D2as_v4<br/>IP privada 10.1.0.6<br/>sin IP pública<br/>Ubuntu 24.04 · Node 24 · forever<br/>FibonacciApp :3000"]
                    end
                end
            end
        end
    end

    users -- "HTTP :80" --> lbip
    lbip --> fe --> rule --> pool
    probe -. "revisa :3000" .-> vm1 & vm2 & vm3
    pool -- "TCP :3000" --> vm1 & vm2 & vm3
    nsg -. "protege las NIC" .- vm1 & vm2 & vm3
    admin -- "SSH :22" --> vm1 & vm2
    vm1 -- "SSH (ProxyJump)" --> vm3
```

* **Usuarios:** entran por HTTP al puerto 80 de la IP pública del balanceador (`20.246.233.87`).
* **Load Balancer:** la regla pasa el tráfico del puerto 80 al 3000 y lo reparte entre las VM del back-end pool que el sondeo marca como sanas.
* **Nodos:** tres VM idénticas en software (Ubuntu 24.04, Node.js 24, `forever` y la FibonacciApp en el puerto 3000), una en cada zona de disponibilidad, dentro de la misma subred y protegidas por el mismo NSG.
* **Administración:** VM1 y VM2 se administran por SSH con su propia IP pública. VM3 no tiene IP pública, así que se llega a ella pasando por VM1 (*jump host*), y sale a Internet a través del balanceador (SNAT).
