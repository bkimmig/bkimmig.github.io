# bkimmig.github.io

Powered by [Hugo](https://github.com/gohugoio/hugo).

# Developing

All the code is kept in this repository. The code for generating the content is
kept on the `source` branch with a github action that pushes the generated site
to the `main` branch.

## Steps to develop

- `git checkout source`
- `git brach -b "your branch"`
- add code
- pull request into source
- merge
- the github action will build and push the code to `main`
