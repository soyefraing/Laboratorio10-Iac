# LABORATIO 10 OBSERVABILIDAD - EFRAIN GASTAÑUADI

## Descripción de la Actividad
Este laboratorio consistió en la implementación y despliegue de un stack de observabilidad centralizado sobre una arquitectura de microservicios en contenedores. El objetivo principal fue configurar la recolección automática de métricas y logs tanto de la infraestructura como de las aplicaciones (frontend y backend) para su análisis en tiempo real. 

---

---

## Comandos de Infraestructura

```bash
# Levantar el stack completo (construye imágenes si faltan)
docker compose up -d --build

# Detener los contenedores manteniendo los volúmenes y dashboards de Grafana
docker compose down

# Reseteo total
docker compose down -v

# Verificar el estado de salud y puertos de todos los contenedores activos
docker compose ps
```

---

## 4. Matriz de Direcciones del Sistema

| Servicio | URL Local | 
| :--- | :--- | 
| **Frontend** | `http://localhost:8080` |
| **Backend** | `http://localhost:3001/metrics` | 
| **Grafana** | `http://localhost:3000` |
| **Prometheus** | `http://localhost:9090` | 

---

## 5. Configuración de Dashboards (Grafana)

### Paneles de Métricas (Prometheus)
* **Panel 1: CPU del Contenedor Backend**
  * **Query:** `sum(rate(node_cpu_seconds_total{mode!="idle"}[1m])) * 100`
  * **Configuración:** Unidad en `Percent (0-100)`, título `"CPU contenedor backend (%)"` y umbral crítico (*Threshold*) en **50** color rojo.
* **Panel 2: CPU del Host**
  * **Query:** `100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[1m])) * 100)`
  * **Configuración:** Unidad en `Percent (0-100)` y título `"CPU del host (%)"`.

### Paneles de Logs (Loki)
* **Panel 3: Logs de la Aplicación**
  * **Query:** `{tier="application"} | json`
  * **Visualización:** Tipo *Logs*, título `"Logs de aplicación (API + frontend)"`.
* **Panel 4: Logs de Infraestructura**
  * **Query:** `{tier="infrastructure"}`
  * **Visualización:** Tipo *Logs*, título `"Logs de infraestructura"`.

*Al finalizar la adición de los componentes, el cuadro de mandos se consolidó bajo el nombre formal:* **Observabilidad - Nombre **.

---

## Configuración de la Regla de Alerta


* **Identificador:** Define el nombre de la regla como `CPU backend > 50%`.
* **Consulta (Query A):**  `sum(rate(node_cpu_seconds_total{mode!="idle"}[1m])) * 100`.
* **Criterio de Activación:** Establece la condición lógica en **IS ABOVE 50** para detectar sobrecostes de procesamiento.
* **Ciclo de Evaluación:** Crea una carpeta contenedora y un grupo de chequeo (*Evaluation group*) con un intervalo de 10s. En el parámetro de Pending period (tiempo de gracia), fija un lapso continuo de 30s para mitigar picos transitorios.

Guarda y consolida la regla haciendo clic en el botón **Save rule and exit**.

---

## Configuración del Punto de Contacto (Webhook)

* **Identificador:** Nombra el punto de enlace como `Webhook Contac`.
* **Tipo de Integración:** Selecciona la opción Webhook.
* **Dirección de Enlace (URL):** Define el endpoint de red interna que procesará las notificaciones en el contenedor: `http://host.docker.internal:3001/alerts`.
* **Validación:** Presiona el botón **Test** para despachar un paquete de prueba controlado y corroborar de forma inmediata la conectividad hacia el backend.

Concluye el registro almacenando los cambios mediante la opción **Save contact point**.

---

## Validación del Ciclo 

Para comprobar el correcto funcionamiento de toda la infraestructura y la respuesta automatizada ante incidentes, sigue estos pasos:

1. **Simulación de Carga:** Accede a la interfaz web del Frontend (`http://localhost:8080`) y presiona el control interactivo **"Generar carga de CPU (30s)"** para iniciar la prueba de estrés en el servidor.
2. **Monitoreo de Métricas:** Dirígete inmediatamente al cuadro de mandos principal en Grafana. Verifica visualmente cómo el panel denominado *"CPU contenedor backend"* experimenta una elevación abrupta, superando el límite crítico del **50%** definido por la línea de umbral.
3. **Ciclo de Vida de la Alerta:** Navega hacia el módulo de **Alerting** -> **Alert rules**. En esta sección podrás auditar la transición de estados de la regla: observará cómo pasa de un estado **Normal** a **Pending** y, una vez transcurrido el tiempo de gracia de 30 segundos continuos, cambia formalmente a **Firing** (Disparada).
4. **Verificación en el Tablero de Logs:** Abre el panel destinado a los registros e historial de logs para validar la recepción de la notificación automática enviada hacia el backend, confirmando el cierre exitoso del flujo de observabilidad.

---

## Respuestas al Cuestionario de Evaluación

### 1. ¿Por qué necesitamos Loki además de Prometheus si ya tenemos `/metrics`?
Porque gestionan dimensiones de datos orientadas a propósitos distintos y complementarios en un entorno de observabilidad:
* **Prometheus** se enfoca de manera exclusiva en datos **cuantitativos** (métricas de series temporales estadísticas como porcentajes, contadores y tasas). Permite identificar *cuándo* y *en qué medida* ocurre una anomalía en el sistema (por ejemplo, detectar que la latencia aumentó o que un servicio está saturado).
* **Loki** se encarga del almacenamiento y consulta de datos **cualitativos** (registros de logs en texto plano). Responde al *por qué* sucedió dicha anomalía, haciendo posible que el equipo de desarrollo inspeccione las trazas de error, excepciones internas y contextos de ejecución exactos en el código de la aplicación.

Disponer únicamente de métricas nos advierte sobre fallos inminentes en la infraestructura, pero sin un motor de logs como Loki resulta inviable ejecutar el diagnóstico o depuración de la causa raíz de un error.

---

### 2. ¿Qué ventaja aporta que las fuentes de datos de Grafana estén aprovisionadas como código y no creadas a mano?
Alinea la plataforma con las prácticas modernas de **GitOps e Infraestructura como Código (IaC)**, ofreciendo las siguientes ventajas clave:
* **Automatización y Repetibilidad:** Permite reproducir e instanciar las conexiones hacia Prometheus y Loki en milisegundos de forma totalmente desatendida al levantar los contenedores, eliminando la necesidad de configurar los parámetros a través de la interfaz gráfica web.
* **Mitigación de Errores Humanos:** Evita inconsistencias de conexión tipográficas, fallos en la declaración de puertos o errores de autenticación comunes durante el despliegue manual.
* **Trazabilidad y Control de Versiones:** Al ser archivos planos de configuración (como YAML), cualquier modificación sobre los endpoints queda registrada y auditada de forma centralizada en el repositorio Git del proyecto, simplificando procesos de integración continua y acelerando los planes de recuperación ante desastres (*Disaster Recovery*).

---

### 3. El panel "CPU contenedor" y el panel "CPU host" pueden mostrar valores muy distintos. ¿Por qué? ¿Cuál usarías para alertar sobre una aplicación concreta?
Difieren de forma notable debido a la escala y el alcance del aislamiento de recursos medidos por cada componente:
* El panel de **CPU del Host** representa la carga agregada de trabajo sobre el hardware o servidor físico global (o máquina virtual), el cual suele contar con múltiples núcleos de procesamiento compartidos entre todos los procesos activos del sistema operativo.
* El panel de **CPU del Contenedor** mide únicamente la cuota relativa de tiempo de procesamiento consumida por el proceso aislado dentro del ciclo de vida de Docker. 

Por este motivo, un contenedor de backend puede encontrarse saturado al 100% de su capacidad nominal y comprometer el rendimiento de su API, mientras que el Host general reporta apenas un 3% de consumo si el servidor subyacente dispone de recursos de sobra.

**Para alertar sobre una aplicación concreta se debe utilizar la CPU del contenedor**, ya que es la única métrica capaz de reflejar fielmente la salud interna y cuellos de botella de ese microservicio específico, sin verse enmascarada por el dimensionamiento general de la máquina.

---

### 4. ¿Qué diferencia hay entre el *evaluation interval* y el *pending period* de una alarma?
Ambos representan etapas cronológicas cruciales dentro de la lógica del ciclo de vida de una regla de alerta, pero operan bajo conceptos muy diferentes:
* **Evaluation interval (Intervalo de evaluación):** Define la frecuencia o tasa de muestreo temporal con la cual el motor de alertas de Grafana ejecuta de forma cíclica la consulta (*query*) sobre la base de datos de Prometheus (por ejemplo, realizar el chequeo del umbral estrictamente cada 10 segundos).
* **Pending period (Periodo de gracia/espera):** Es un filtro de estabilización temporal que actúa una vez que la métrica ha cruzado el límite crítico. La alarma entra provisionalmente en estado *Pending*; si la métrica se mantiene infringiendo el umbral establecido de forma ininterrumpida durante la totalidad de ese tiempo asignado (por ejemplo, 30 segundos continuos), la alerta cambia su estado formal a *Firing* y despacha las notificaciones por el webhook.

El intervalo de evaluación controla la tasa de muestreo, mientras que el periodo *Pending* sirve como amortiguador para prevenir falsos positivos provocados por picos breves o ráfagas transitorias normales en los recursos de los servidores.
