# Puerta de Validación — OMS FreshDirect

## Checklist de Especificación

### 1. Requisitos
- [x] Contexto y alcance definidos (incluyendo lista explícita de "Fuera de alcance")
- [x] Stakeholders identificados con sus intereses (8 stakeholders)
- [x] Requisitos funcionales documentados (formato Actor+SHALL+Acción+Resultado), todos priorizados, sin ambigüedad, sin mezclar diseño (50 FR)
- [x] Requisitos no-funcionales con SLO medible y método de medición (13 NFR)
- [x] Requisitos regulatorios mapeados a normativa concreta (7 REG)
- [x] Requisitos operativos definidos (CI/CD, monitorización, DR, runbooks) (6 OPS)
- [x] Glosario completo (16 términos)

### 2. Casos de Uso y Criterios de Aceptación
- [x] Casos de uso cubren todos los FR críticos, con flujos alternativos (8 casos de uso, UC-01 a UC-08)
- [x] Escenarios BDD para FRs críticos (feliz + error + borde) (10 features Gherkin)
- [x] Criterios EARS para NFR/REG/OPS (los 5 patrones usados donde corresponda) (26 criterios EARS)
- [x] No hay criterios ambiguos o no verificables

### 3. Especificaciones
- [x] Spec funcional: 14 endpoints con input/output/errores tipados, mapeados a FR
- [x] Spec estructural: arquitectura, dependencias, modelo de datos (14 entidades), infraestructura justificada
- [x] Spec comportamental: máquina de estados de Pedido, comportamiento ante fallo (5 dependencias), timeouts/reintentos, SLI/SLO/SLA
- [x] Spec operativa: CI/CD, métricas/alertas (9 reglas), 2 runbooks completos, backup/DR, logging

### 4. Contratos Formales
- [x] Operaciones críticas (`confirmarPedido`, `escanearProducto`) tienen precondiciones, postcondiciones e invariantes explícitos
- [x] Invariantes de dominio identificados (idempotencia de pago, FEFO, no sobreventa)
- [x] Tipos de entrada/salida especificados, incluida función pura `calcularTotalesConIVA`

### 5. ADRs
- [x] 5 decisiones arquitectónicas documentadas (base de datos, arquitectura general, caché/consistencia de stock, pasarela de pago, optimización de rutas)
- [x] Cada ADR incluye contexto, decisión, alternativas y consecuencias (positivas y negativas)

### 6. Resiliencia
- [x] Patrones de resiliencia asignados por dependencia externa (9 dependencias en YAML)
- [x] Jerarquía de degradación y feature flags definidos (3 niveles, 5 flags)
- [x] Error budget definido (temporada normal y temporada alta)

### 7. Trazabilidad
- [x] Matriz bidireccional sin referencias huérfanas (FR, NFR, REG, OPS cubiertos)
- [x] Cadena de derivación documentada (5 ejemplos concretos del dominio)

## Observaciones y Supuestos Documentados

Dado que el archivo fuente (`entrevistas_stakeholders.md`) es una fuente completa y detallada, no hubo necesidad de inventar requisitos de negocio. Sin embargo, se documentan aquí los supuestos técnicos añadidos para completar la especificación, ya que el archivo fuente no baja a nivel de implementación:

1. **Supuesto:** el mecanismo exacto de idempotencia de pago (idempotency keys generadas en frontend) no se especifica en las entrevistas; se asume como práctica estándar de la industria para cumplir el requisito explícito de "nada de cobrar dos veces" (Roberto Sánchez). *Riesgo:* bajo, es un patrón estándar de Stripe.
2. **Supuesto:** el umbral de "margen de al menos 3 días" para aplicar FEFO se infiere de combinar dos datos de entrevistas distintas (política de no enviar productos con caducidad ≤ hoy+2 días, y el principio general de FEFO). *Riesgo:* bajo, pero debe confirmarse con Ana Martínez antes de implementación.
3. **Supuesto:** el mecanismo concreto de failover (RDS Multi-AZ) y el dimensionamiento del Auto Scaling Group no se mencionan explícitamente en las entrevistas; se derivan de los NFR de disponibilidad y de la restricción explícita "EC2 + RDS, no serverless ni Kubernetes". *Riesgo:* bajo — coherente con el mandato del CEO.
4. **Riesgo abierto (no supuesto, gap real):** las entrevistas no especifican qué ocurre si un cliente reporta una incidencia sobre un pedido con más de una línea afectada simultáneamente (ej. dos productos dañados en el mismo pedido). Se recomienda aclarar con Lucía Fernández si el límite de "2 reembolsos automáticos al mes" cuenta por incidencia o por línea.
5. **Riesgo abierto:** no se especifica el comportamiento exacto cuando un pedido dividido por peso (FR-050, >25kg) sufre una entrega fallida en solo uno de los dos envíos. Se recomienda aclarar con Pablo Ruiz antes de la implementación de picking/rutas.
6. **Conflicto potencial ya señalado en 01-Requirements.md:** tensión entre NFR-005 (propagación de stock <5s) y NFR-001/NFR-002 (rendimiento bajo carga de pico), resuelta arquitectónicamente en ADR-003; y entre NFR-007 (minimizar datos en logs) y la necesidad de observabilidad de fallos de pago, resuelta con enmascarado de campos sensibles.

Ninguno de estos supuestos o riesgos abiertos bloquea el gate, pero se recomienda resolverlos como acción de seguimiento antes de iniciar el desarrollo de los módulos afectados (Incidencias y Rutas/Picking respectivamente).

```
═══════════════════════════════════
  GATE DE VALIDACIÓN — RESULTADO
═══════════════════════════════════
  Requisitos:            PASS (50 FR + 13 NFR + 7 REG + 6 OPS)
  Criterios aceptación:  PASS (10 Features BDD + 26 criterios EARS)
  Spec funcional:        PASS
  Spec estructural:      PASS
  Spec comportamental:   PASS
  Spec operativa:        PASS
  Contratos:             PASS
  ADRs:                  PASS (5 ADRs)
  Resiliencia:           PASS
  Trazabilidad:          PASS
  ─────────────────────────────────
  DECISIÓN:  🟡 GO CONDICIONAL
  ─────────────────────────────────
```

## Criterios de Decisión

| Resultado | Condición |
|---|---|
| ✅ GO | Todos los checkpoints pasan sin observaciones abiertas |
| 🟡 GO CONDICIONAL | 1-2 checkpoints menores fallan o quedan gaps no bloqueantes, con acción de mitigación identificada |
| ❌ NO-GO | Fallan checkpoints críticos (ej. ausencia de contratos en operaciones críticas, ADRs faltantes, referencias huérfanas en trazabilidad) |

**Justificación de GO CONDICIONAL:** todos los checkpoints formales del checklist pasan (✅ en las 7 secciones). La calificación condicional se debe exclusivamente a los dos riesgos abiertos documentados arriba (puntos 4 y 5), que son preguntas de negocio pendientes de aclarar con Lucía Fernández y Pablo Ruiz, no defectos de la especificación en sí. **Acción recomendada antes de iniciar desarrollo:** agendar una sesión de aclaración de 30 minutos con ambos stakeholders para resolver los dos gaps antes de comenzar la implementación de los módulos de Incidencias y de Picking/Rutas; el resto del sistema puede iniciar desarrollo sin bloqueo.
