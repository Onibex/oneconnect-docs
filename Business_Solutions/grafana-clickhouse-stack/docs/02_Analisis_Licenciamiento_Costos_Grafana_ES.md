# Análisis de Licenciamiento, Costos y Recomendación — Grafana sobre ClickHouse

**Versión:** 1.2 · **Idioma:** Español · **Verificado a:** julio 2026
**Escenario de referencia:** aprox. 15 usuarios · ClickHouse ya desplegado y accesible por red · Objetivo: dashboards rápidos y de bajo costo sobre ClickHouse.

> **Documento complementario:** la implementación técnica (instalación con Docker Compose, data source de ClickHouse, TLS/nginx) se detalla en el manual `01_Manual_Integracion_Grafana_ClickHouse_ES.md`.

---

## Resumen y recomendación

**Recomendamos Grafana OSS auto-hospedado, en la misma red o región que el ClickHouse existente.**

El análisis se sustenta en tres razones:

1. **El costo de licencia es USD 0**, tanto para 15 como para 150 usuarios: Grafana OSS es gratuito bajo AGPLv3, y este es el factor determinante.
2. **El conector de ClickHouse es oficial y gratuito.** No hace falta Grafana Enterprise para conectarse a ClickHouse. Enterprise solo agrega valor para SAML nativo, RBAC fino, reportes PDF programados y soporte con SLA — ninguno obligatorio aquí.
3. **Latencia mínima y camino de datos privado.** Un Grafana en la misma red que ClickHouse lo consulta por **IP privada**, sin salir a Internet. Grafana Cloud también es viable en cuanto a velocidad, pues corre en AWS/GCP con solo decenas de milisegundos de latencia adicional, de modo que la ventaja de estar dentro de la misma red es real pero moderada. El diferenciador determinante sigue siendo el costo de licencia (USD 0), no la velocidad.

**Cuándo reconsiderar la recomendación:** si el cliente exige contractualmente soporte con SLA de Grafana Labs, SAML corporativo estricto, o reportes PDF programados automáticos → evaluar **Grafana Cloud Pro** (aprox. USD 115/mes para 15 usuarios) o **Enterprise** (cotización). Ver Sección 8.

**El número de usuarios (15) no cambia la recomendación**, solo permite poner cifras concretas a la opción Cloud, que es la única sensible al conteo de usuarios.

---

## 1. Modelos de despliegue y licenciamiento

Grafana se ofrece en tres formas. Es importante no confundir "Cloud" con "de pago": OSS puede ser auto-hospedado gratis, y Cloud tiene un nivel gratuito.

| | **Grafana OSS** | **Grafana Cloud** | **Grafana Enterprise (self-managed)** |
|---|---|---|---|
| Dónde corre | Servidor propio / EC2 | SaaS de Grafana Labs | Servidor propio / EC2 |
| Licencia | AGPLv3, **gratis** | Free / Pro / Enterprise (Advanced) | Comercial, **de pago** |
| Usuarios | **Ilimitados, gratis** | Se cobra por usuario activo (salvo 3 gratis) | Licencia por usuario/instancia |
| Operación (parches, backups, TLS) | A cargo del cliente | A cargo de Grafana Labs | A cargo del cliente |
| Features Enterprise (SAML, RBAC, reportes) | No nativas | Sí (según nivel) | Sí |
| Conector ClickHouse | Sí (gratis) | Sí (gratis) | Sí |

**Licencia AGPLv3 (aclaración):** desde abril 2021 el núcleo de Grafana es AGPLv3. Esto **permite el uso comercial e interno** y desplegar el binario sin modificar. La obligación de publicar el código fuente solo aplica si se *modifica* Grafana y se *ofrece como servicio a terceros por red* — no es el caso de dashboards internos. Para un despliegue de dashboards internos, AGPLv3 no representa una restricción práctica.

---

## 2. ¿ClickHouse obliga a comprar Enterprise? — No

Las "**fuentes de datos premium**" que Grafana reserva para Enterprise son conectores propietarios como **Splunk, Oracle, ServiceNow, New Relic, Datadog, Snowflake, Dynatrace**, etc. **ClickHouse no está en esa lista**: su plugin es oficial, open-source, firmado por Grafana Labs y gratuito, y funciona en OSS.

**Conclusión:** la fuente de datos elegida (ClickHouse) es 100% compatible con la edición gratuita. Este es el punto que descarta Enterprise como requisito técnico.

---

## 3. Costos de Grafana Cloud

Grafana Cloud factura por **consumo** (métricas/logs/trazas ingeridas) **+ por usuario activo de visualización**. Niveles oficiales:

| Nivel | Precio base | Qué incluye | Retención |
|---|---|---|---|
| **Free** | **USD 0** (para siempre) | Todos los servicios con uso limitado; **3 usuarios activos**; 10k series de métricas; 50 GB logs/trazas/perfiles; soporte comunidad | 14 días |
| **Pro** | **Desde USD 19/mes** + consumo | Pago por uso sobre el Free; soporte email 8×5 | 13 meses métricas / 30 días logs-trazas |
| **Enterprise (Advanced)** | **Desde USD 25.000/año** de compromiso | SAML, RBAC, auditoría, fuentes premium, SLA, despliegue flexible (nube pública/federal/BYOC) | Personalizada |

**Detalle clave:** en un despliegue de BI que solo consulta un ClickHouse existente —sin ingerir métricas, logs ni trazas en Grafana Cloud— los medidores de consumo (series, GB) casi no aplican, y los límites de retención de 14 días del Free resultan irrelevantes, ya que los datos y su retención los gestiona ClickHouse y no Grafana. En la práctica, el costo de Cloud en este escenario se reduce a la tarifa por usuario activo más la cuota de plataforma.

**Tarifa por usuario (Pro):** aproximadamente **USD 8 al mes por cada usuario activo de visualización** por encima de los 3 gratuitos, más una cuota de plataforma de **USD 19 al mes** (según fuentes de mercado verificadas a junio 2026). Grafana reestructuró su pricing hacia consumo, por lo que **conviene confirmar la tarifa exacta por usuario en la calculadora oficial** (`grafana.com/pricing`) antes de cotizar en firme.

> El "Free" de Cloud está limitado a **3 usuarios activos**. Con 15 usuarios el nivel Free **no alcanza**; habría que ir a Pro.

> **Conectividad:** si el ClickHouse es accesible desde Internet, Grafana Cloud puede conectarse **directamente** a su endpoint (con TLS y un firewall/Security Group restringido a los rangos de Grafana Cloud), sin necesidad de **Private Data Source Connect (PDC)**. Si se prefiere no exponer el puerto, PDC es la opción más segura. Contrapartida a evaluar: mantener un endpoint de base de datos accesible desde Internet, aunque esté restringido por IP.

---

## 4. Costos de Grafana Enterprise y variación entre self-managed y Cloud

**Grafana Enterprise (self-managed)** — la edición comercial que se instala en servidor propio:

- **Precio no público**: se cotiza con ventas. Los reportes de la comunidad varían mucho: desde aprox. **USD 4.000–5.000/año** (mediana reportada por compradores pequeños) hasta el punto de partida de lista de aprox. **USD 25.000/año**, con descuentos negociables del 20–35% en compromisos anuales.
- Es una licencia (por usuario/instancia) que se **suma** al costo de la infraestructura y operación propias (servidor, backups, personal). No incluye hosting.

**Grafana Cloud Enterprise (Advanced)** — las mismas features Enterprise pero en el SaaS:

- Parte de un **compromiso mínimo de USD 25.000/año** + consumo. Reduce la carga operativa (no se administran servidores) pero agrega los costos de ingesta por uso.

**El costo sí varía entre Cloud y on-premise:**
- **Self-managed:** a la licencia cotizada se suman la infraestructura y el personal propios; ofrece más control a cambio de más trabajo operativo.
- **Cloud:** un compromiso anual más el consumo, sin trabajo de infraestructura; implica menos control, pero también menos operación.
- Para 15 usuarios haciendo BI sobre ClickHouse, **ninguna de las dos se justifica económicamente** frente a OSS, salvo que se requieran las features Enterprise específicas de Sección 5.

---

## 5. OSS vs Enterprise/de pago — matriz y pros/contras

### Qué agrega Enterprise (Cloud Advanced o self-managed) sobre OSS

| Función | OSS | Enterprise | ¿Relevante para 15 usuarios / ClickHouse? |
|---|---|---|---|
| Visualización, paneles, alertas | Completo | Sí | — (OSS ya lo cubre) |
| Conector ClickHouse | Sí | Sí | **OSS suficiente** |
| Usuarios ilimitados | Sí | Sí | a favor de OSS |
| SSO OAuth/OIDC (Google, Azure AD, Keycloak) | Sí | Sí | OSS lo soporta|
| **SAML nativo** | (emulable con oauth2-proxy) | Sí | Solo si es requisito corporativo estricto |
| **RBAC de grano fino** (roles personalizados) | (solo Viewer/Editor/Admin por carpeta/equipo) | Sí | Con 15 usuarios, los roles básicos suelen bastar |
| Permisos por fuente de datos | No | Sí | Normalmente innecesario aquí |
| **Reportes PDF programados por email** | No | Sí | **Único gap real** si el cliente los pide |
| Caché de consultas | No | Sí | Deseable con muchos usuarios; ClickHouse ya es rápido |
| Branding personalizado / white-label | No | Sí | Cosmético |
| Auditoría / usage insights | No | Sí | Solo si hay exigencias de cumplimiento |
| Fuentes premium (Splunk, Oracle, etc.) | No | Sí | No aplica (usan ClickHouse) |
| **Soporte con SLA de Grafana Labs** | (comunidad) | Sí | Depende de tolerancia al riesgo del cliente |

### Pros y contras

**Grafana OSS**

**Ventajas:**
- Costo de licencia USD 0; usuarios ilimitados.
- Control total, datos y despliegue en la propia infraestructura (bueno para gobierno de datos).
- Mismo motor de visualización y el mismo conector ClickHouse que las ediciones de pago.
- La opción más rápida para ClickHouse on-prem (sin salto a Internet).

**Limitaciones:**
- Toda la operación es propia: parches, backups, TLS, disponibilidad.
- Sin SAML nativo (hay que usar OIDC o un proxy), sin RBAC fino, sin reportes PDF programados.
- Soporte por comunidad, sin SLA.

**Grafana Enterprise / de pago (Cloud o self-managed)**

**Ventajas:**
- SAML, RBAC fino, reportes PDF programados, auditoría, caché de consultas.
- Soporte con SLA (respaldo del proveedor).
- Cloud elimina la carga operativa.

**Limitaciones:**
- Costo relevante (USD miles a decenas de miles al año); mínimos altos en Cloud Enterprise (USD 25k/año).
- Cloud contra ClickHouse on-prem requiere túnel (PDC) y agrega latencia de red.
- Se paga por funciones que este tipo de uso mayormente no necesita.

---

## 6. Escenario de 15 usuarios — comparación de costos

Supuestos: 15 usuarios de visualización, ClickHouse on-prem ya existente (costo cero adicional), uso de BI (sin ingesta de telemetría en Grafana).

### Costo anual de licencia/suscripción (solo software)

| Opción | Cálculo | **Costo anual (licencia)** |
|---|---|---|
| **OSS auto-hospedado** | Gratis, cualquier nº de usuarios | **USD 0** |
| **Grafana Cloud Free** | Limitado a 3 usuarios → **no alcanza** para 15 | No viable |
| **Grafana Cloud Pro** | (USD 19 plataforma + 12 usuarios × aprox. USD 8) × 12 meses | **aprox. USD 1.380** |
| **Grafana Cloud Advanced** | aprox. 15 × USD 55/mes × 12 | aprox. USD 9.900 (mín. compromiso USD 25k/año) → **≥USD 25.000** |
| **Enterprise self-managed** | Cotización comercial | **aprox. USD 5.000–25.000+** |

### Costo total estimado 1er año (TCO: incluye infraestructura y operación)

| Opción | Licencia | Infra (servidor) | Operación / setup | **TCO 1er año (aprox.)** |
|---|---|---|---|---|
| **OSS auto-hospedado** | USD 0 | aprox. USD 200–400 (EC2 t3.medium) | Horas del equipo para instalar y operar | **aprox. USD 200–400 + horas internas** |
| **Grafana Cloud Pro** | aprox. USD 1.380 | USD 0 (SaaS) | Conexión directa a ClickHouse (TLS + Security Group); PDC opcional | **aprox. USD 1.380 + setup** |
| **Enterprise self-managed** | aprox. USD 5.000–25.000+ | aprox. USD 200–400 | Setup + operación (interno) | **aprox. USD 5.200–25.400+** |

**Conclusión:** para 15 usuarios, OSS es dramáticamente más barato en licencia (USD 0). El costo real de OSS es el **esfuerzo de implementación y operación** (instalación, actualizaciones, backups). Cloud Pro es una alternativa razonable (aprox. USD 1.380/año) **solo si** se prefiere no administrar servidores; si el ClickHouse es accesible desde Internet, la conexión desde Cloud es directa (sin PDC). Enterprise no se justifica salvo requisitos específicos de Sección 5.

---

## 7. Consideraciones y costos "ocultos" del despliegue self-hosted

Estos puntos deben cotizarse aparte porque no son licencia de Grafana pero sí costo real del proyecto:

| Ítem | Detalle | Costo típico |
|---|---|---|
| **Servidor** | Máquina modesta (2 vCPU, 4–8 GB, SSD), p. ej. un `t3.medium`–`t3.large` en AWS, en la **misma red/región** que el ClickHouse | aprox. USD 15–35/mes on-demand; menos con planes de ahorro |
| **Dominio** | Subdominio propio (p. ej. `grafana.tu-dominio.com`) | Si ya tienen dominio, USD 0; nuevo dominio aprox. USD 10–15/año |
| **DNS / IP fija** | Registro `A` + Elastic IP en AWS | Elastic IP asociada: sin costo; sin asociar: menor |
| **Certificado SSL** | **nginx (en Docker Compose) con certificado propio** (bring-your-own-cert); alternativa: **Let's Encrypt gratis** con Caddy o certbot | USD 0 |
| **Seguridad del ClickHouse** | Si el endpoint es accesible desde Internet: firewall/Security Group restringido (nunca `0.0.0.0/0`), TLS activo y usuario de solo lectura; idealmente Grafana lo consume por **red privada interna** | USD 0 (configuración) |
| **Base de config** | **SQLite** (por defecto) es suficiente y estable para aprox. 15 usuarios; PostgreSQL solo si se requiere HA | USD 0 (SQLite) · RDS aprox. USD 15–30/mes solo si HA |
| **Backups** | Respaldo del volumen `storage` (SQLite `grafana.db`) + dashboards como código | Menor |
| **Operación** | Parches, monitoreo, actualizaciones | Horas del equipo (parte del servicio) |
| **Alta disponibilidad (opcional)** | 2ª instancia de Grafana + Postgres compartido + balanceador | Solo si el cliente exige HA |

Para 15 usuarios internos, un despliegue de **una sola instancia** con **Docker Compose** (Grafana OSS + nginx/TLS, SQLite) sobre un EC2 `t3.medium` es suficiente y de bajo costo.

---

## 8. Recomendación final y árbol de decisión

**Recomendación:** **Grafana OSS auto-hospedado** (desplegado con **Docker Compose**: Grafana + nginx TLS + SQLite), co-ubicado con ClickHouse. Licencia USD 0, máxima velocidad, control total de datos.

**Reconsiderar hacia una edición de pago solo si se cumple alguno de estos disparadores:**

- ¿El cliente **exige soporte con SLA** del proveedor? → Enterprise o Cloud.
- ¿Requiere **SAML corporativo nativo** (no OIDC)? → Enterprise/Cloud (o emular con oauth2-proxy en OSS).
- ¿Necesita **reportes PDF programados y enviados por email** automáticamente? → Enterprise/Cloud (único gap funcional real de OSS para BI).
- ¿Requiere **RBAC de grano fino** o auditoría por cumplimiento normativo? → Enterprise/Cloud.
- ¿**No quiere administrar servidores** en absoluto? → **Grafana Cloud Pro** (aprox. USD 115/mes para 15 usuarios). Como el ClickHouse es público, la conexión es directa (TLS + Security Group a rangos de Cloud); PDC opcional si no se quiere exponer el endpoint.

Si **ninguno** aplica → **OSS** es la respuesta correcta, y los 15 usuarios lo confirman: es la única opción donde el conteo de usuarios no genera costo.

> **Configuración de referencia:** Grafana OSS self-hosted con **Docker Compose** (Grafana + nginx TLS + SQLite) y el data source de ClickHouse por HTTP. El detalle está en el manual `01_Manual_Integracion_Grafana_ClickHouse_ES.md`.
---

## 9. Fuentes y notas de verificación

- Página oficial de precios de Grafana Cloud (`grafana.com/pricing`): Free USD 0, Pro desde USD 19/mes + consumo, Enterprise desde USD 25.000/año.
- Tarifa por usuario activo (aprox. USD 8/mes sobre 3 gratis en Pro): fuentes de mercado verificadas a junio 2026; confirmar en la calculadora oficial dado el cambio a pricing por consumo.
- Features exclusivas de Enterprise (SAML, RBAC, reportes, fuentes premium, caché, branding, usage insights): documentación oficial de Grafana Enterprise.
- Plugin ClickHouse: oficial, open-source, firmado por Grafana Labs, gratuito; funciona en OSS y Cloud.
- Precios de Enterprise self-managed: no públicos; rangos según reportes de compradores (mediana aprox. USD 4.800/año) y lista de partida (aprox. USD 25k/año), negociables.
- Licencia AGPLv3 desde abril 2021; uso comercial e interno permitido.

*Todos los precios y capacidades corresponden a julio 2026 y pueden cambiar. Las cifras de Enterprise y la tarifa por usuario son estimaciones de mercado, no cotizaciones en firme; para un número contractual solicitar cotización a Grafana Labs y validar la calculadora oficial.*
