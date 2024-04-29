# LeseApp

A simple app to help kids learn to read.

Try it from mobile at [https://janmechtel.github.io/leseapp/](https://janmechtel.github.io/leseapp/)

1. Take a picture of the text to be read
2. Text is read out loud 
3. Kid is asked to read parts of the text (Coming soon)

## Roadmap
- [x] stop recording after 5s

- [ ] compare transcription to text + otherwise try again celebrate success
- [ ] UI progress bar during recording
- [ ] try again on error
- [ ] instructions (first written, ideally spoken)
- [ ] cache the debugging audio / transcriptions to not always call the cloud


## Setup
- using Google Cloud Vision for OCR and Google Speech for Text to Speech
- using simple Google Cloud Function to provide temporary access tokens to the front end
- using Github Pages for Hosting


## Local Development

- npm install
- npm run dev

## Deploy
- `npm run deploy`


## Recommended IDE Setup

[VSCode](https://code.visualstudio.com/) + [Volar](https://marketplace.visualstudio.com/items?itemName=Vue.volar) (and disable Vetur) + [TypeScript Vue Plugin (Volar)](https://marketplace.visualstudio.com/items?itemName=Vue.vscode-typescript-vue-plugin).

## Type Support for `.vue` Imports in TS

TypeScript cannot handle type information for `.vue` imports by default, so we replace the `tsc` CLI with `vue-tsc` for type checking. In editors, we need [TypeScript Vue Plugin (Volar)](https://marketplace.visualstudio.com/items?itemName=Vue.vscode-typescript-vue-plugin) to make the TypeScript language service aware of `.vue` types.

## Customize configuration

See [Vite Configuration Reference](https://vitejs.dev/config/).

## Project Setup

```sh
npm install
```

### Compile and Hot-Reload for Development

```sh
npm run dev
```

### Type-Check, Compile and Minify for Production

```sh
npm run build
```

### Lint with [ESLint](https://eslint.org/)

```sh
npm run lint
```
