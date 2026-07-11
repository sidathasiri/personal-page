# Repository for Personal Web Page Content

## Setting Up

1. `git submodule sync`
2. `git submodule update --init --recursive --remote`
3. `cd sidathasiri`
4. `npm install`

## Build static files

`cd sidathasiri && hugo -t port-hugo`

## Deploy

1. `cd sidathasiri/public`
2. Add, commit, and push: `git push origin main`
3. `cd ..`
4. Add, commit, and push parent repo changes: `git push origin main`