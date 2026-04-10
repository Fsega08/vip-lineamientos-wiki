# Glosario

[Volver al indice](/)

---

## Terminos del Dominio Catastral

| Termino | Definicion |
|---------|-----------|
| **LADM-COL** | Land Administration Domain Model — Colombia. Estandar colombiano de catastro multiproposito, version 4.1. Define las entidades, relaciones y nomenclaturas del dominio catastral |
| **NPN** | Numero Predial Nacional. Identificador unico de 30 digitos asignado por IGAC a cada predio en Colombia. Formato: `{depto}{muni}{zona}{sector}{manzana}{predio}{mejora}` |
| **Predio** | Unidad inmobiliaria con identidad juridica, fisica y economica. Identificado por su NPN |
| **Avaluo Catastral** | Valor oficial asignado a un predio por la autoridad catastral, base para el impuesto predial |
| **Mutacion Catastral** | Cambio en la informacion de un predio: transferencia de propiedad, subdivision, englobamiento, actualizacion de datos |
| **Interviniente** | Persona natural o juridica relacionada con un predio (propietario, poseedor, usufructuario) |
| **Geometria** | Representacion espacial de un predio en coordenadas geograficas. Se almacena y valida con PostGIS |

## Entidades Externas

| Sigla | Nombre Completo | Rol |
|-------|-----------------|-----|
| **IGAC** | Instituto Geografico Agustin Codazzi | Autoridad catastral nacional. Provee datos de predios, NPN, cartografia |
| **SNR** | Superintendencia de Notariado y Registro | Registro de propiedad, tradicion y libertad de inmuebles |
| **BIAN** | Banking Industry Architecture Network | Framework de arquitectura bancaria. VIP adopta algunos patrones de nomenclatura de BIAN para la estructura de `Datos_Operacion` |

## Terminos de la Plataforma VIP

| Termino | Definicion |
|---------|-----------|
| **VIP** | Vortex Integration Platform. Plataforma de interoperabilidad (middleware) entre consumidores y backends |
| **Canonical** | Estructura de mensaje estandar (request/response) que todos los servicios VIP deben usar. Ver [Canonical](contratos/canonical.md) |
| **Capa de interoperabilidad** | Conjunto de microservicios que median entre consumidores externos y backends internos/externos. No contienen logica de negocio propia, traducen y enrutan |
| **Adaptador** | Componente que traduce la comunicacion entre VIP y un backend externo (IGAC, SNR). Mapea formatos, errores y protocolos |
| **Prefijo** | Identificador configurable del proyecto. Para Alcaldia de Medellin es `acm`. Se usa en nombres de microservicios, queues, keys de Redis y namespaces |
| **Datos_Operacion** | Seccion del Canonical que contiene los datos especificos de cada servicio. Su estructura varia por endpoint |

## Terminos Tecnicos

| Termino | Definicion |
|---------|-----------|
| **Kong** | API Gateway open-source. Punto de entrada unico a VIP. Maneja autenticacion, rate limiting, CORS y enrutamiento |
| **Keycloak** | Servidor de identidad open-source. Maneja autenticacion OIDC, emision de JWT, roles y permisos |
| **Vault** | Gestor de secretos de HashiCorp. Almacena passwords, tokens y certificados de forma segura |
| **OIDC** | OpenID Connect. Protocolo de autenticacion sobre OAuth 2.0, usado entre Kong y Keycloak |
| **JWT** | JSON Web Token. Token firmado que contiene claims del usuario (roles, permisos, expiracion) |
| **PostGIS** | Extension espacial de PostgreSQL. Permite almacenar y consultar geometrias, validar coordenadas |
| **KEDA** | Kubernetes Event-Driven Autoscaling. Escala pods basado en metricas de colas RabbitMQ o requests HTTP |
| **DLQ** | Dead Letter Queue. Cola donde van los mensajes que fallaron despues de los reintentos maximos (3 reintentos, backoff 1s/2s/4s, TTL 24h) |
| **IRSA** | IAM Roles for Service Accounts. Mecanismo de AWS para que pods de K8s asuman roles de IAM sin credenciales estaticas |
| **HPA** | Horizontal Pod Autoscaler. Escala automaticamente el numero de pods basado en CPU/memoria |
| **JSON Pointer** | Notacion estandar (RFC 6901) para referenciar campos dentro de un JSON. Ejemplo: `/Datos_Operacion/Numero_Documento`. Se usa en el campo `Ruta` de los errores |
| **Snake_Case** | Convencion de nomenclatura donde las palabras se separan con guion bajo y cada palabra inicia con mayuscula. Ejemplo: `Fecha_Hora_Solicitud`. Adoptada por VIP alineada con LADM-COL |

## Codigos de Error — Prefijos

| Prefijo | Dominio |
|---------|---------|
| **VLD** | Validacion — estructura, formato, campos requeridos |
| **BIZ** | Negocio General — logica de negocio generica |
| **CTR** | Catastro — predios, mutaciones, avaluos |
| **GEO** | Geoespacial — geometrias, PostGIS, coordenadas |
| **SEC** | Seguridad — autenticacion y autorizacion |
| **INT** | Integracion — comunicacion con servicios externos |
| **SYS** | Sistema — infraestructura, errores internos |
| **URB** | Urbanismo — licencias urbanisticas *(futuro)* |
| **HAC** | Hacienda — impuesto predial, tributario *(futuro)* |
