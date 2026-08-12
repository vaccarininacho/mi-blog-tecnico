---
title: "De monolito a servicios desacoplados: lo que aprendimos escalando nuestro sistema de pedidos"
date: 2026-08-11
tags: [arquitectura, backend, post-mortem, aprendizajes]
---

# De monolito a servicios desacoplados: lo que aprendimos escalando nuestro sistema de pedidos

## Contexto

Trabajaba en el equipo backend de una plataforma de e-commerce mediana, con un monolito en Node.js (Express) y PostgreSQL que manejaba catálogo, carrito, pagos y logística en un mismo proceso. El equipo tenía 6 desarrolladores y el sistema procesaba alrededor de 15.000 pedidos diarios en temporada normal. Durante campañas de descuentos, el tráfico se multiplicaba por 8, y ahí empezaron los problemas.

## Problema

Cada Black Friday, el módulo de pedidos se volvía el cuello de botella: una sola tabla de `orders` con múltiples triggers y validaciones síncronas (stock, pagos, notificaciones, facturación) generaba bloqueos de escritura y tiempos de respuesta de más de 6 segundos por pedido. En la campaña más crítica, el sistema completo cayó durante 40 minutos porque el proceso de facturación síncrona saturó las conexiones a la base de datos, arrastrando con él al carrito y al login.

El desafío de arquitectura era claro: un solo servicio con responsabilidades acopladas no podía escalar de forma independiente, y cualquier fallo en un submódulo (facturación) tumbaba funcionalidades que no tenían relación directa (login).

## Acciones

1. **Post-mortem constructivo**: organizamos una sesión sin culpas 48 horas después del incidente, con toda la evidencia de logs y métricas sobre la mesa. Documentamos la cadena exacta de fallos: pedido → validación de stock → facturación síncrona → timeout → saturación del pool de conexiones → caída del login.
2. **Identificación de límites de dominio**: separamos las responsabilidades en tres candidatos a servicio: `orders-core`, `billing` y `notifications`, priorizando desacoplar primero lo que había causado la caída (facturación).
3. **Migración incremental con Strangler Fig**: en lugar de una reescritura completa (que el equipo había propuesto inicialmente), extrajimos `billing` como servicio independiente con una cola de mensajes (RabbitMQ) para procesar facturación de forma asíncrona, manteniendo el monolito como orquestador temporal.
4. **Pruebas de carga previas a la siguiente campaña**: simulamos 8x el tráfico normal en un entorno de staging antes de la siguiente fecha crítica, usando k6.
5. **Documentación y control de versiones**: cada cambio se trabajó en ramas dedicadas con PRs revisados por al menos dos personas del equipo, y cada decisión de arquitectura quedó registrada en un ADR (Architecture Decision Record) dentro del repositorio.

## Aprendizajes

- **Desacoplar no significa migrar todo de una vez**: extraer un solo servicio crítico (facturación) redujo el riesgo y nos dio resultados medibles rápido, sin la incertidumbre de una reescritura total.
- **La resiliencia se diseña, no se improvisa**: mover la facturación a un flujo asíncrono evitó que un fallo en un submódulo tumbara servicios no relacionados.
- **Los post-mortems sin culpas generan mejor información**: cuando el foco estuvo en el sistema y no en las personas, el equipo compartió detalles que de otra forma se habrían omitido por miedo a señalamientos.
- **Las pruebas de carga deben simular el peor escenario real**, no un promedio optimista; la campaña siguiente resistió el pico de tráfico sin caídas.

## Reflexión sobre feedback radicalmente sincero

Durante el post-mortem, un compañero senior cuestionó directamente mi propuesta inicial de reescribir el sistema completo, señalando que subestimaba el riesgo y el tiempo que tomaría, y que probablemente estaba motivada más por el deseo de "empezar de cero" que por una evaluación objetiva del problema. Ese feedback fue incómodo de escuchar en el momento, pero fue el que nos llevó a optar por la migración incremental con el patrón Strangler Fig, que terminó siendo la decisión correcta: entregamos valor medible (el servicio de facturación desacoplado) en tres semanas, en lugar de arriesgar meses en una reescritura completa. Aprendí que el feedback directo y sin filtros, cuando viene con evidencia y buena intención, es una de las formas más rápidas de corregir el rumbo antes de que un error de juicio se vuelva costoso.

---

*Enlaces a evidencia de control de versiones:*
- PR de extracción del servicio de facturación: `https://github.com/vaccarininacho/mi-blog-tecnico/pull/XX`
- ADR documentando la decisión de arquitectura: `https://github.com/vaccarininacho/mi-blog-tecnico/blob/main/docs/adr/001-billing-service-extraction.md`
- Commit con los resultados de las pruebas de carga: `https://github.com/vaccarininacho/mi-blog-tecnico/commit/XXXXXXX`
