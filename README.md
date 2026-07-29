# React + TypeScript + Vite

This template provides a minimal setup to get React working in Vite with HMR and some ESLint rules.

## CI/CD pipelines

Four workflows, split across a trust boundary. Code from a pull request builds
**without any credentials**; the privileged deploy/cleanup steps run from
definitions on `main`, so a PR cannot edit the job that receives AWS access.

```
                         ┌─────────────────────────────────────────────┐
                         │   UNTRUSTED (PR code)      │  TRUSTED (main) │
                         └─────────────────────────────────────────────┘

═ A: PR opened / updated ══════════════════════════════════════════════════════

  PR push          Pipeline (pull_request)          Preview lifecycle
  ───────►   ┌──────────────────────────┐    ┌──────────────────────────────┐
             │ ci  → CI (reusable)      │    │ job: deploy                  │
             │      checks + build      │    │  if: name==Pipeline &&       │
             │      ▲ upload artifact   │    │      event==pull_request &&  │
             │        "build"           │    │      conclusion==success &&  │
             │ deploy → SKIPPED         │    │      same-repo (not a fork)  │
             │   (only push to main)    │    │                              │
             └───────────┬──────────────┘    │  ⏸  pause: approve "dev"     │
                         │ green CI          │  → assume gha-dev-preview    │
                         └──workflow_run─────►    (OIDC, previews/* only)   │
                            (completed)       │  → download artifact "build"│
                                              │  → validate (never execute) │
                                              │  → s3 sync → previews/pr-N/  │
                                              └──────────────┬───────────────┘
                                                             ▼
                                       s3://previews/pr-N/  →  CloudFront dev
                                       https://…cloudfront.net/previews/pr-N/

═ B: PR closed ════════════════════════════════════════════════════════════════

  PR closed      Preview cleanup request           Preview lifecycle
  ───────►   ┌──────────────────────────┐    ┌──────────────────────────────┐
             │ permissions: {}          │    │ job: cleanup                 │
             │ echo only (no code,      │    │  if: name==Preview cleanup.. │
             │  no AWS, no secrets)     │    │      && success && same-repo │
             └───────────┬──────────────┘    │  → PR number from head_branch│
                         │ completed         │    (API) + integer validation│
                         └──workflow_run─────►  ⏸  pause: approve "dev"     │
                                              │  → assume gha-dev-preview    │
                                              │  → s3 rm previews/pr-N/      │
                                              │  → prove: prefix empty +     │
                                              │           URL 403/404        │
                                              └──────────────────────────────┘

═ C: merge to main (production) ═══════════════════════════════════════════════

  push main        Pipeline (push)
  ───────►   ┌──────────────────────────┐
             │ ci  → CI (build)         │
             │ deploy (if: push)        │
             │  ⏸ approve "production"  │  → s3://…assets-1/ (prod bucket)
             │  → gha-production-deploy │  → invalidate CloudFront prod
             │     (OIDC)               │
             └──────────────────────────┘
   (Preview lifecycle does NOT fire — event==push, not pull_request)
```

**Workflows**

| File                                           | Trigger                            | Role                                                                     |
| ---------------------------------------------- | ---------------------------------- | ------------------------------------------------------------------------ |
| `pipeline.yaml` (`Pipeline`)                   | `pull_request`, `push`, `dispatch` | CI on every PR; production deploy on push to `main`                      |
| `ci.yaml` (`CI`, reusable)                     | `workflow_call`                    | lint, format, build, upload `build` artifact — no credentials            |
| `preview-cleanup-request.yaml`                 | `pull_request: closed`             | inert signal (`permissions: {}`) so the trusted side can observe a close |
| `preview-lifecycle.yaml` (`Preview lifecycle`) | `workflow_run: completed`          | trusted deploy + cleanup of `previews/pr-N/` via the dev OIDC role       |

**Why it is safe**

1. **Trust boundary** — PR code builds with no credentials; deploy/cleanup run from `main`, so a PR cannot alter the privileged job it triggers.
2. **Separate OIDC roles** — `gha-dev-preview` (scoped to `previews/*`) vs `gha-production-deploy` (production only). A compromised preview cannot reach production.
3. **`same-repo` guard in every `if:`** — a fork gets CI but no dev identity. "Forks receive no secrets" is not enough, because OIDC is not a stored secret.

Currently, two official plugins are available:

- [@vitejs/plugin-react](https://github.com/vitejs/vite-plugin-react/blob/main/packages/plugin-react) uses [Oxc](https://oxc.rs)
- [@vitejs/plugin-react-swc](https://github.com/vitejs/vite-plugin-react/blob/main/packages/plugin-react-swc) uses [SWC](https://swc.rs/)

## React Compiler

The React Compiler is not enabled on this template because of its impact on dev & build performances. To add it, see [this documentation](https://react.dev/learn/react-compiler/installation).

## Expanding the ESLint configuration

If you are developing a production application, we recommend updating the configuration to enable type-aware lint rules:

```js
export default defineConfig([
  globalIgnores(["dist"]),
  {
    files: ["**/*.{ts,tsx}"],
    extends: [
      // Other configs...

      // Remove tseslint.configs.recommended and replace with this
      tseslint.configs.recommendedTypeChecked,
      // Alternatively, use this for stricter rules
      tseslint.configs.strictTypeChecked,
      // Optionally, add this for stylistic rules
      tseslint.configs.stylisticTypeChecked,

      // Other configs...
    ],
    languageOptions: {
      parserOptions: {
        project: ["./tsconfig.node.json", "./tsconfig.app.json"],
        tsconfigRootDir: import.meta.dirname,
      },
      // other options...
    },
  },
]);
```

You can also install [eslint-plugin-react-x](https://github.com/Rel1cx/eslint-react/tree/main/packages/plugins/eslint-plugin-react-x) and [eslint-plugin-react-dom](https://github.com/Rel1cx/eslint-react/tree/main/packages/plugins/eslint-plugin-react-dom) for React-specific lint rules:

```js
// eslint.config.js
import reactX from "eslint-plugin-react-x";
import reactDom from "eslint-plugin-react-dom";

export default defineConfig([
  globalIgnores(["dist"]),
  {
    files: ["**/*.{ts,tsx}"],
    extends: [
      // Other configs...
      // Enable lint rules for React
      reactX.configs["recommended-typescript"],
      // Enable lint rules for React DOM
      reactDom.configs.recommended,
    ],
    languageOptions: {
      parserOptions: {
        project: ["./tsconfig.node.json", "./tsconfig.app.json"],
        tsconfigRootDir: import.meta.dirname,
      },
      // other options...
    },
  },
]);
```
