# LinkedIn Post Templates

## Fórmula Ganadora: Error → Aprendizaje → Solución → Pregunta

```
[Hook — error o subestimación]

[Contexto — qué pasó, 2-3 líneas]

Lo que aprendí:
• [lección 1]
• [lección 2]
• [lección 3]

[Pregunta / CTA]
```

### Ejemplo concreto:

> Subestimé los tests unitarios durante mucho tiempo.
>
> Arrancaba proyectos, codeaba rápido, y "después veía los tests". Después nunca llegaban.
>
> Hasta que un bug en producción me obligó a reverlo. Ahora:
>
> • Tests primero → menos bugs, duermo mejor
> • Testeo la lógica de negocio, no los frameworks
> • El feedback loop es instantáneo
>
> ¿Qué hábito de testing les costó adoptar?

---

## Fórmula: Cambio de Opinión

```
[Hook — cambié de opinión]

[Qué pensaba antes — 2 líneas]

[Qué pasó que me hizo cambiar — experiencia real]

[Qué pienso ahora — con humildad]

[Pregunta]
```

### Ejemplo concreto:

> Cambié de opinión sobre Clean Architecture.
>
> Cuando la vi por primera vez pensé: "muchas carpetas al pedo, overengineering puro".
>
> Pero después de mantener un proyecto sin arquitectura clara —6 meses después era imposible tocar nada sin romper todo— entendí el punto.
>
> No es overengineering. Es **inversión en el futuro del proyecto**.
>
> ¿Alguna tecnología/patrón que hayan despreciado y después abrazado?

---

## Fórmula: Problema → Solución (3 puntos)

```
[Hook — problema que resolviste]

[Contexto rápido]

3 problemas que resolví con [tecnología]:
• [problema + solución 1]
• [problema + solución 2]
• [problema + solución 3]

[Pregunta]
```

### Ejemplo concreto:

> 3 problemas que Docker me resolvió:
>
> • Setup local inconsistente — cada dev tenía versions distintas
> • Base de datos reproducible — testeo con la misma DB que producción
> • Onboarding más rápido — git clone + docker compose up y listo
>
> ¿Qué herramienta les cambió el día a día?

---

## Fórmula: Decisión Técnica

```
[Hook — decisión con criterio]

[Contexto — qué necesitaba resolver]

[Decisión — qué elegí y por qué]
[Lo que no elegí y por qué no]

[Cierre con aprendizaje]

[Pregunta]
```

### Ejemplo concreto:

> Elegí modular monolith en lugar de microservicios. Y no me arrepiento.
>
> El proyecto era un sistema de gestión de empleo: órdenes, postulantes, pagos, encuestas. Dominios relacionados pero separados.
>
> Microservicios me daban:
> • + escalabilidad (que no necesitaba)
> • + complejidad operativa (que no quería)
> • - cohesión en el equipo chico
>
> Modular monolith me dio:
> • Un código base organizado por dominio
> • Despliegue simple
> • Poder partir después si hace falta
>
> La mejor arquitectura es la que resolve el problema de HOY, no el que imaginás para dentro de 2 años.
>
> ¿Monolith o microservicios? ¿Cómo deciden?

---

## Serie: Backend Notes #N

```
[Hook]

[Contenido]

—

Backend Notes #{N} — [tema del día]

[CTA]
```

### Ejemplo concreto - Backend Notes #1:

> JWT: la implementación más simple no es suficiente.
>
> Arranqué con el middleware clásico: verificar token, decodificar, next(). Después:
>
> • No tenía refresh tokens → sesiones cortas, usuarios enojados
> • No separaba auth de autorización → middleware único que hacía TODO
> • Los tests eran una pesadilla
>
> Lo separe en 3 capas:
>
> 1. **Auth**: verifica el token
> 2. **Authorization**: checkea permisos
> 3. **Business rules**: la lógica post-auth
>
> Testing se volvió trivial. Cada capa tiene una responsabilidad.
>
> —
>
> Backend Notes #1 — JWT bien hecho
>
> ¿Cómo estructuran la autenticación en sus proyectos?

---

## Fórmula: Reflexión / Aprendizaje

```
[Hook — reflexión honesta]

[Lo que pasó — narrativa personal]

[Lo que aprendiste]

[Pregunta abierta]
```

### Ejemplo concreto:

> Hay una diferencia enorme entre "escribir código" y "construir software".
>
> Pasar de uno a otro me llevó:
> • Escribir tests antes que código
> • Pensar en mantenimiento antes que features
> • Aceptar que lo que no está documentado no existe
>
> ¿En qué momento sintieron que pasaron de codear a construir?

---

## Quick Reference: Hooks Fuertes

| ❌ Mal | ✅ Bien |
|--------|---------|
| Hoy quiero hablar de testing | Subestimé los tests unitarios durante mucho tiempo |
| Docker es una herramienta muy útil | 3 problemas que resolví usando Docker |
| Últimamente he estado aprendiendo | Hay una diferencia enorme entre escribir código y construir software |
| En este post les voy a contar | Cambié de opinión sobre Clean Architecture |
| Quiero compartir mi experiencia | Cometí un error diseñando una API que me costó horas |

## Quick Reference: CTAs Efectivos

| Tipo | Ejemplo |
|------|---------|
| Experiencia | ¿Les pasó? / ¿Cómo lo resuelven? |
| Opinión | ¿Qué opinan? / ¿Qué hubieran hecho distinto? |
| Aporte | ¿Qué agregarían? / ¿Cómo lo estructuran ustedes? |
| Reflexión | ¿En qué momento sintieron que...? |
| Hábito | ¿Qué hábito les costó adoptar? |
