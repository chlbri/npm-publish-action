# NPM Publish Action

A GitHub composite action that simplifies NPM package publishing with smart version checking.

## 🚀 Features

- 🧠 **Smart** : Only publishes if the version number in `package.json` differs from the latest on NPM
- 🛠 **Configurable** : Customize the version checking behavior, registry URL, and package path
- 🔐 **Secure** : Keeps your NPM authentication token secret
- ⚡ **Fast** : Based on the proven `JS-DevTools/npm-publish@v3` action
- 📤 **Detailed outputs** : Exposes old and new version numbers, and the type of change

## 📋 Usage

### Basic usage

```yaml
name: Publish to NPM

on:
  push:
    branches: [main]

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - run: npm test
      
      - name: Publish to NPM
        uses: chlbri/npm-publish-action@v1
        with:
          token: ${{ secrets.NPM_TOKEN }}
```

### Advanced usage

```yaml
- name: Publish to NPM
  uses: chlbri/npm-publish-action@v1
  with:
    token: ${{ secrets.NPM_TOKEN }}
    registry: https://registry.npmjs.org/
    package: ./packages/my-package
    tag: latest
    access: public
    provenance: true
    strategy: upgrade
    ignore-scripts: true
    dry-run: false
```

### Publishing to GitHub Package Registry

```yaml
- name: Publish to GitHub Packages
  uses: chlbri/npm-publish-action@v1
  with:
    token: ${{ secrets.GITHUB_TOKEN }}
    registry: https://npm.pkg.github.com
    access: public
```

## 📖 Input parameters

| Name | Type | Default | Description |
|-----|------|--------|-------------|
| `token` | string | **required** | Authentication token to use with the configured registry |
| `registry` | string | `https://registry.npmjs.org/` | Registry URL to use |
| `package` | string | Current directory | Path to a package directory, a `package.json`, or a packed `.tgz` to publish |
| `tag` | string | `latest` | Distribution tag for publishing |
| `access` | string | NPM defaults | Package visibility (`public` or `restricted`) |
| `provenance` | boolean | `false` | Run `npm publish` with the `--provenance` flag |
| `strategy` | string | `all` | Publishing strategy (`all` or `upgrade`) |
| `ignore-scripts` | boolean | `true` | Run `npm publish` with the `--ignore-scripts` flag |
| `dry-run` | boolean | `false` | Run `npm publish` with the `--dry-run` flag |

### Publishing strategies

- **`all`** (default) : Publishes any version that does not yet exist in the registry
- **`upgrade`** : Publishes only if the version is a semver upgrade of the requested `tag`

## 📤 Outputs

This action exposes several output variables that you can use in subsequent steps of your workflow:

```yaml
- name: Publish to NPM
  id: publish
  uses: chlbri/npm-publish-action@v1
  with:
    token: ${{ secrets.NPM_TOKEN }}

- name: Use outputs
  if: ${{ steps.publish.outputs.type }}
  run: |
    echo "Package published: ${{ steps.publish.outputs.id }}"
    echo "Release type: ${{ steps.publish.outputs.type }}"
```

| Name | Type | Description |
|-----|------|-------------|
| `id` | string | Package identifier: `${name}@${version}` or empty if no release |
| `type` | string | Semver release type, `initial` if first release, `different` if other change, or empty if no release |
| `name` | string | Package name |
| `version` | string | Package version |
| `old-version` | string | Previously published version on the `tag` or empty if no previous version |
| `tag` | string | Distribution tag the package was published to |
| `access` | string | Access level the package was published with |
| `registry` | string | Registry the package was published to |
| `dry-run` | boolean | Whether `npm publish` was run in "dry run" mode |

## 🔧 Exemples d'utilisation

### Workflow complet avec build et tests

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - run: npm ci
      - run: npm run build
      - run: npm test
      - run: npm run lint

  publish:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - run: npm ci
      - run: npm run build
      
      - name: Publish to NPM
        id: publish
        uses: chlbri/npm-publish-action@v1
        with:
          token: ${{ secrets.NPM_TOKEN }}
          strategy: upgrade
          provenance: true
      
      - name: Create GitHub Release
        if: ${{ steps.publish.outputs.type }}
        uses: actions/create-release@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          tag_name: v${{ steps.publish.outputs.version }}
          release_name: Release ${{ steps.publish.outputs.version }}
          body: |
            Package ${{ steps.publish.outputs.name }} has been updated from ${{ steps.publish.outputs.old-version }} to ${{ steps.publish.outputs.version }}.
            
            Release type: ${{ steps.publish.outputs.type }}
```

### Workflow avec monorepo

```yaml
name: Publish Packages

on:
  push:
    branches: [main]

jobs:
  publish:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        package:
          - packages/core
          - packages/utils
          - packages/cli
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      
      - run: npm ci
      - run: npm run build --workspace=${{ matrix.package }}
      
      - name: Publish ${{ matrix.package }}
        uses: chlbri/npm-publish-action@v1
        with:
          token: ${{ secrets.NPM_TOKEN }}
          package: ${{ matrix.package }}
          access: public
```

## 🛠 Développement

Cette action est basée sur [`JS-DevTools/npm-publish@v3`](https://github.com/JS-DevTools/npm-publish) et fournit une interface simplifiée avec des valeurs par défaut sensées.

### Structure du projet

```
npm-publish-action/
├── action.yml          # Définition de l'action
├── README.md          # Documentation
└── LICENSE            # Licence
```

## 📝 Configuration requise

- **Token NPM** : Vous devez configurer un secret `NPM_TOKEN` dans votre repository avec votre token d'authentification NPM
- **Permissions** : Pour publier vers GitHub Packages, assurez-vous que `GITHUB_TOKEN` a les permissions `packages: write`

## 🔗 Liens utiles

- [Documentation officielle des GitHub Actions](https://docs.github.com/en/actions)
- [Création et visualisation des tokens d'authentification NPM](https://docs.npmjs.com/creating-and-viewing-authentication-tokens)
- [Travail avec le registre NPM de GitHub Packages](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-npm-registry)

## 📄 Licence

MIT License - voir le fichier [LICENSE](LICENSE) pour plus de détails.

## 🙏 Remerciements

Cette action GitHub est construite au-dessus de l'excellent travail de **James Messinger** et de l'équipe [JS-DevTools](https://github.com/JS-DevTools). 

Un grand merci à [@JamesMessinger](https://github.com/JamesMessinger) pour avoir créé et maintenu l'action [`npm-publish`](https://github.com/JS-DevTools/npm-publish) qui constitue le cœur de cette action composite.

## 🤝 Contribution

Les contributions sont les bienvenues ! N'hésitez pas à ouvrir une issue ou un pull request.

---

Créé avec ❤️ par [chlbri](https://github.com/chlbri)