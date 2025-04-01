Hugo Commands
=========

Create new post

```sh
hugo new post/<postName>.md
```

Serve locally with drafts

```sh
hugo serve -D
```

Build regular page

```sh
hugo
```

This will also update the files in the `public` submodule. The CI will automatically update
the `gh-pages` branch, which will contain the static website.
