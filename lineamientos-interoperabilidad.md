# Lineamientos de Interoperabilidad Catastro

[Volver al indice](index.md)

---

## 1. Objetivo

Este documento establece los lineamientos, convenciones y practicas estandar que rigen el diseno, desarrollo e integracion de todos los microservicios de la capa de interoperabilidad de la plataforma VIP (Vortex Integration Platform), implementada para la Alcaldia de Medellin en el contexto del dominio catastral LADM-COL 4.1.

El proposito central es garantizar que los servicios de interoperabilidad sean coherentes, trazables, seguros y alineados con los modelos de datos y nomenclaturas del estandar colombiano de catastro multiproposito.

### Aplica a

- Todos los microservicios de la capa middleware de VIP.
- Los adaptadores hacia backends externos (IGAC, SNR, Hacienda Municipal).
- Los servicios geoespaciales apoyados en PostGIS.
- La gestion de identidad y autorizacion con Keycloak/OIDC.
- Cualquier nuevo microservicio que se integre en el futuro a la plataforma VIP.

### Beneficios

- Reducir la ambiguedad en el diseno de contratos de API.
- Garantizar respuestas homogeneas y predecibles para los consumidores de los servicios.
- Facilitar la trazabilidad de extremo a extremo en arquitecturas distribuidas con multiples integraciones.
- Proteger la informacion sensible evitando la exposicion de detalles tecnicos internos en las respuestas de error.
- Acelerar la incorporacion de nuevos modulos mediante convenciones ya establecidas.

---

## 2. Canonical

El Canonical define la estructura de mensaje estandar que deben utilizar todos los servicios de la plataforma VIP, tanto en solicitudes como en respuestas. Esta estructura garantiza uniformidad en el contrato de API independientemente del dominio o backend subyacente.

> **Nota:** aunque los campos sean opcionales, la estructura completa siempre debe enviarse; los campos vacios se transmiten con valor nulo o vacio segun su tipo de dato.

Ver documento completo: [Canonical de Interoperabilidad](canonical.md)

---

## 3. Manejo de Errores

Este lineamiento define las estrategias estandar para la gestion de errores en todos los microservicios de la capa de interoperabilidad VIP. El objetivo es garantizar respuestas coherentes, trazables y alineadas con el modelo LADM-COL, sin exponer informacion tecnica interna al consumidor.

Ver documento completo: [Manejo de Errores](manejo-errores.md)

---

## 4. Estandares de Nomenclatura de Microservicios y Colas

Ver documento completo: [Nombramiento de Microservicios y Colas](nombramiento-microservicios-colas.md)

---

## 5. Lineamientos para el Manejo de Cache

Ver documento completo: [Estandar de Cache Redis](cache-redis.md)

---

## 6. Guia de Documentacion de Servicios

Ver documento completo: [Guia de Documentacion de Servicios](guia-documentacion.md)
