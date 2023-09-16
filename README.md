# Repository for Personal Web Page Content

## Setting Up

1. `git submodule sync`
2. `git submodule update --init --recursive --remote`

## Build static files

`hugo -t hugo-theme-cleanwhite`

## Deploy

1. `cd public`
2. Add, commit and push `git push origin main`
3. cd ..
4. Add, commit and push `git push origin main`