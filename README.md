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

This will also update the files in the `public` submodule. To update the blog, checkout
`main` inside the submodule first, then build the page and commit and push the newly generated
website.
