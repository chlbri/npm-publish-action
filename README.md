# NPM Publish Action

Une action GitHub composite qui simplifie la publication de packages NPM avec vérification intelligente des versions.

## 🚀 Fonctionnalités

- 🧠 **Intelligent** : Publie uniquement si la version dans `package.json` diffère de la dernière version sur NPM
- 🛠 **Configurable** : Personnalisez le comportement de vérification des versions, l'URL du registre et le chemin de votre package
- 🔐 **Sécurisé** : Garde votre token d'authentification NPM secret
- ⚡ **Rapide** : Basé sur l'action éprouvée `JS-DevTools/npm-publish@v3`
- 📤 **Sorties détaillées** : Expose les anciens et nouveaux numéros de version, et le type de changement

## 📋 Utilisation

### Utilisation basique

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

### Utilisation avancée

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

### Publication vers GitHub Package Registry

```yaml
- name: Publish to GitHub Packages
  uses: chlbri/npm-publish-action@v1
  with:
    token: ${{ secrets.GITHUB_TOKEN }}
    registry: https://npm.pkg.github.com
    access: public
```

## 📖 Paramètres d'entrée

| Nom | Type | Défaut | Description |
|-----|------|--------|-------------|
| `token` | string | **requis** | Token d'authentification à utiliser avec le registre configuré |
| `registry` | string | `https://registry.npmjs.org/` | URL du registre à utiliser |
| `package` | string | Répertoire courant | Chemin vers un répertoire de package, un `package.json`, ou un `.tgz` à publier |
| `tag` | string | `latest` | Tag de distribution pour la publication |
| `access` | string | Défauts NPM | Visibilité du package (`public` ou `restricted`) |
| `provenance` | boolean | `false` | Exécuter `npm publish` avec le flag `--provenance` |
| `strategy` | string | `all` | Stratégie de publication (`all` ou `upgrade`) |
| `ignore-scripts` | boolean | `true` | Exécuter `npm publish` avec le flag `--ignore-scripts` |
| `dry-run` | boolean | `false` | Exécuter `npm publish` avec le flag `--dry-run` |

### Stratégies de publication

- **`all`** (défaut) : Publie toute version qui n'existe pas encore dans le registre
- **`upgrade`** : Publie uniquement si la version est une mise à niveau semver du `tag` demandé

## 📤 Sorties

Cette action expose plusieurs variables de sortie que vous pouvez utiliser dans les étapes suivantes de votre workflow :

```yaml
- name: Publish to NPM
  id: publish
  uses: chlbri/npm-publish-action@v1
  with:
    token: ${{ secrets.NPM_TOKEN }}

- name: Utiliser les sorties
  if: ${{ steps.publish.outputs.type }}
  run: |
    echo "Package publié : ${{ steps.publish.outputs.id }}"
    echo "Type de release : ${{ steps.publish.outputs.type }}"
```

| Nom | Type | Description |
|-----|------|-------------|
| `id` | string | Identifiant du package : `${name}@${version}` ou vide si pas de release |
| `type` | string | Type de release semver, `initial` si première release, `different` si autre changement, ou vide si pas de release |
| `name` | string | Nom du package |
| `version` | string | Version du package |
| `old-version` | string | Version précédemment publiée sur le `tag` ou vide si aucune version précédente |
| `tag` | string | Tag de distribution vers lequel le package a été publié |
| `access` | string | Niveau d'accès avec lequel le package a été publié |
| `registry` | string | Registre vers lequel le package a été publié |
| `dry-run` | boolean | Si `npm publish` a été exécuté en mode "dry run" |

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