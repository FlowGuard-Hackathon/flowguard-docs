# Convención de ramas

Convención para nombrar ramas en todos los repositorios de `FlowGuard-Hackathon`. Los mensajes de commit tienen su propia convención en [convencion-commits.md](convencion-commits.md).

## Formato

```
<tipo>/<puesto>-<descripcion-corta>
```

| Parte | Regla | Ejemplo |
|---|---|---|
| `tipo` | Los mismos tipos que los commits: `feat`, `fix`, `docs`, `refactor`, `test`, `chore`, `perf`, `ci`, `build` | `feat` |
| `puesto` | Puesto de quien abre la rama: `dev1` a `dev5` | `dev3` |
| `descripcion-corta` | En español, minúsculas, palabras separadas con guion, sin acentos ni ñ, de 2 a 5 palabras | `herramientas-lectura` |

Ejemplos:

- `feat/dev3-herramientas-lectura`
- `fix/dev1-umbral-pag-001`
- `docs/dev5-contratos-pago`
- `ci/dev4-typecheck-en-pr`
- `refactor/dev3-renombrado-glosario`

El nombre completo no supera los 50 caracteres. Sin espacios, mayúsculas, acentos, ñ ni caracteres especiales distintos de `/` y `-`.

## Reglas

1. **`main` es la única rama permanente.** Nadie trabaja ni hace push directo sobre `main`; todo entra por Pull Request revisado.
2. **Una rama, un PR, un objetivo.** Si el trabajo crece, se divide en otra rama.
3. **Ramas cortas.** Se crean desde `main` actualizado (`git pull` antes de `git checkout -b`) y se fusionan en uno o dos días como máximo.
4. **Antes de abrir el PR,** actualiza tu rama con `main` y vuelve a correr las comprobaciones del repositorio (por ejemplo `npm test` y `npm run typecheck`).
5. **Tras fusionar, se borra la rama** en GitHub y en local (`git branch -d <rama>`).
6. **Trabajo de otro puesto:** si una rama toca rutas de otro propietario, se acuerda antes con él (ver la propiedad por rutas en el [README](README.md)).

## Comandos

```bash
git checkout main
git pull
git checkout -b feat/dev3-herramientas-lectura
# ...trabajo y commits según convencion-commits.md...
git push -u origin feat/dev3-herramientas-lectura
gh pr create --base main --head feat/dev3-herramientas-lectura
```

## Ramas anteriores a esta convención

Las ramas creadas antes de esta convención (`dev3/agent-skeleton`, `docs/five-positions-*`, `chore/*-skeleton`, `docs/readme-agent-guide`, `chore/gitignore-secrets`) ya están fusionadas y se conservan como historial. La rama `dev` del repositorio `flowguard` no tiene cambios respecto de `main` y queda fuera de uso. La rama local `dev3/read-tools`, que todavía no se ha publicado, se renombrará a `feat/dev3-herramientas-lectura` al adaptarla al glosario.
