# Convención de commits

Todos los repositorios de `FlowGuard-Hackathon` usan **Conventional Commits** con el texto en español. Las ramas tienen su propia convención en [convencion-ramas.md](convencion-ramas.md).

## Formato

```
<tipo>(<alcance>): <descripción>

<cuerpo opcional>

<pie opcional>
```

- **tipo:** uno de la tabla de abajo, en minúsculas y en inglés (es el estándar que reconocen las herramientas).
- **alcance:** opcional, en español y en minúsculas; indica el módulo afectado (ver tabla de alcances).
- **descripción:** en español, en imperativo y en minúscula inicial, sin punto final, de 72 caracteres como máximo. Ejemplo: `agrega`, `corrige`, `elimina`, no `agregado` ni `agregando`.
- **cuerpo:** opcional. Explica el **porqué** del cambio, no el cómo. Se separa con una línea en blanco.
- **pie:** opcional. `BREAKING CHANGE: <qué se rompe>` para cambios incompatibles y `Refs: #<número>` para enlazar issues o PRs.

## Tipos

| Tipo | Cuándo se usa | Ejemplo |
|---|---|---|
| `feat` | Funcionalidad nueva | `feat(herramientas): agrega obtener_evidencia con validación de parámetros` |
| `fix` | Corrección de un error | `fix(reglas): evalúa PAG-005 en centavos y no en pesos` |
| `docs` | Solo documentación | `docs(contratos): documenta el error EJECUCION_NO_ENCONTRADA` |
| `refactor` | Cambio interno sin alterar el comportamiento | `refactor(agente): separa la validación del informe del proveedor` |
| `test` | Agregar o corregir pruebas | `test(herramientas): cubre evidencia de otra ejecución` |
| `chore` | Mantenimiento, configuración o dependencias | `chore: actualiza zod a 3.25` |
| `style` | Formato, sin cambios de lógica | `style(app): aplica formato al panel` |
| `perf` | Mejora de rendimiento | `perf(persistencia): agrega índice por ejecucion_id` |
| `ci` | Workflows de CI/CD | `ci: ejecuta typecheck y pruebas en cada PR` |
| `build` | Sistema de build o empaquetado | `build(app): ajusta la configuración de Next.js` |
| `revert` | Revierte un commit anterior | `revert: feat(herramientas): agrega obtener_evidencia` |

## Alcances sugeridos

| Alcance | Módulo | Puesto propietario |
|---|---|---|
| `agente` | `src/servidor/agente/` | Dev 3 |
| `herramientas` | `src/servidor/herramientas/` | Dev 3 |
| `reglas` | Motor de reglas | Dev 1 |
| `ejecuciones` | Orquestación de ejecuciones | Dev 1 |
| `persistencia` | Acceso a datos | Dev 1 |
| `bd` | Migraciones SQL (`flowguard-supabase`) | Dev 1 |
| `contratos` | Esquemas Zod compartidos y `contratos-api` | Dev 5 |
| `mocks` | Fixtures (`flowguard-mocks`) | Dev 5 |
| `pruebas` | Suite compartida (`flowguard-tests`) | Dev 5 coordina |
| `app` | Frontend (`flowguard-app`) | Dev 2 |
| `ci` | Workflows | Dev 4 |
| `glosario`, `planificacion`, `organizacion` | Documentación general | Compartido |

Los nombres de carpetas siguen el [glosario](glosario.md). Mientras el renombrado no termine, el alcance usa el nombre nuevo aunque la carpeta todavía tenga el viejo.

## Reglas

1. **Un commit, un cambio lógico.** No mezcles una funcionalidad con un renombrado masivo o con cambios de formato.
2. **El código y las pruebas del cambio van en el mismo commit**, o en commits seguidos dentro del mismo PR.
3. **Cambios incompatibles** (renombrar un campo de la API, una columna o un código de error) llevan `!` después del tipo y un pie `BREAKING CHANGE:`. Ejemplo: `feat(contratos)!: renombra execution_id a ejecucion_id`.
4. **Sin firmas de herramientas de IA.** No se añaden líneas `Co-Authored-By` de asistentes ni menciones a herramientas de IA en commits ni en PRs.
5. **Sin secretos.** Nunca subas `.env`, claves ni contraseñas, ni siquiera en el texto del commit.
6. **Identidad.** Cada quien firma con su cuenta de GitHub del equipo. Antes del primer commit en un clon nuevo, comprueba la identidad con `git log -1 --format="%an <%ae>"`.

## Ejemplos completos

```
feat(herramientas): implementa las cinco herramientas de lectura

Las herramientas dependen de la interfaz FuenteDatosHerramientas para que
Dev 1 conecte la persistencia real sin tocar este módulo.

Refs: #5
```

```
feat(contratos)!: renombra los identificadores al español

BREAKING CHANGE: todos los campos de la API cambian según glosario.md;
los clientes deben actualizarse en el mismo bloque.
```

## Títulos de Pull Request

El título del PR sigue el mismo formato que un commit, por ejemplo `feat(herramientas): herramientas de lectura del agente`. Si el PR se fusiona con *squash*, ese título se convierte en el commit final.
