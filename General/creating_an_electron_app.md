<meta name="daria:article_id" content="creating_an_electron_app">
<meta name="daria:title" content="Creating an electron app">
<meta name="daria:title_slug" content="creating_an_electron_app">
<meta name="daria:order" content="2">
<meta name="daria:created_on" content="2024-07-07">
<meta name="daria:tags" content="">
<meta name="daria:image_id" content="space">

# Setting up an Electron app

This article will go over setting up an Electron app with the following:

* [Typescript](https://www.typescriptlang.org/)
* [React](https://react.dev/)
* [Tailwind](https://tailwindcss.com/)
* [Web pack](https://webpack.js.org/)
* [ui/shadcn](https://ui.shadcn.com/)

This is by no means a definitive guild, 
but I wanted to create a reference for my own future use.

I will be using `npm` for this. 

## Create the app

Start off by creating the Electron app using [electron-forge](https://www.electronforge.io/):

```shell
npm init electron-app@latest [project-name] -- --template=webpack-typescript
```

## Install React

Run the following to install React:

```shell
cd [project-name]
npm install --save react react-dom
npm install --save-dev @types/react @types/react-dom
```

Then we can create a `app.tsx` file. 
This can go anywhere, but it makes most sense to put it in the `src` directory.

At this point I like to move to my IDE of choice (WebStorm).

```tsx
import './index.css';

import * as React from 'react';
import { createRoot} from "react-dom/client";

const root = createRoot(document.getElementById('root'));

root.render(
    <React.StrictMode>
        <h1>Hello, World!</h1>
    </React.StrictMode>
)
```

This will have some errors, but they should be fixable by updating the `tsconfig.json` file:

```json
{
  "compilerOptions": {
    ...
    "jsx": "react" <- Add this
  },
  ...
}
```

WebStorm will give you option to apply this as a fix automatically.

Now in `src/renderer.ts` you can add a reference to `app.tsx`:

```ts
import './index.css';
import './app' // <- Add this
```

Finally you can now update `src/index.html` to render the app:

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8" />
    <title>Hello World!</title>

  </head>
  <body>
    <div id="root"></div>
  </body>
</html>
```

Now you can run the app with the following command:

```shell
npm start
```

## Installing Tailwind

In the project directory run:

```shell
npm install tailwindcss autoprefixer --save-dev
```

Then run: 

```shell
npx tailwindcss init
```

In the newly created `tailwind.config.js` file update the content:

```js
/** @type {import('tailwindcss').Config} */
module.exports = {
  content: [
    './src/**/*.{ts,tsx,html}' // <- Add this
  ],
  theme: {
    extend: {},
  },
  plugins: [],
}
```

Now install `postcss` and `postcss-loader`:

```shell
npm install postcss postcss-loader
```

Create a `postcss.config.js` file and add this:

```js
import tailwindcss from 'tailwindcss'
import autoprefixer from 'autoprefixer'
export default {
  plugins: [tailwindcss('./tailwind.config.js'), autoprefixer]
}
```

Add a rule to `webpack.renderer.config.ts`:

```ts
rules.push({
    test: /\.css$/,
    use: [{loader: 'style-loader'},
        {loader: 'css-loader'},
        {loader: 'postcss-loader'} // <- add this
    ],
});
```

Finally, you can add `tailwind` directives to the `src/index.css` file:

```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

Now you should be able to add `tailwind` classes to your components:

```tsx
// src/app.tsx

import './index.css';

import * as React from 'react';
import { createRoot} from "react-dom/client";

const root = createRoot(document.getElementById('root'));

root.render(
    <React.StrictMode>
        <h1 className="font-bold text-2xl underline font-mono">Hello, World!</h1>
    </React.StrictMode>
)
```

## Set up ui/shadcn

Now we will set up `ui/shadcn` so we can easily use high quality components.

Run: 

```json
{
  "compilerOptions": {
    ...
    "paths": {
      "*": ["node_modules/*"],
      "@/*": [ "./src/*" ] <- Add this
    },
    ...
  },
  ...
}
```

```shell
npx shadcn-ui@latest init
```



