---
## Software Information
title: Setting Up Next.js with Tailwind CSS
date: 2023-02-08
description: A guide on how to set up a Next.js project with Tailwind CSS.
icon: nextjs.svg

## Author Information
author: Bartek Paczesny (@j5on)
---

[Next.js](https://nextjs.org) is a popular React framework for building modern web applications, and [Tailwind CSS](https://tailwindcss.com) is a utility-first CSS framework for rapidly building custom designs. This guide will walk you through setting up a new Next.js project with Tailwind CSS.

## Prerequisites

-   Node.js and npm installed on your machine
-   Basic knowledge of JavaScript and React

## Setting Up Next.js

First, let's create a new Next.js application. Open your terminal and run the following command:

```sh
npx create-next-app@latest my-next-app
```

Replace my-next-app with the name of your project. This command will create a new directory with the specified name, and it will set up a new Next.js application in that directory.

## Installing Tailwind CSS

Next, navigate into your new project's directory:

```sh
cd my-next-app
```

Then, install Tailwind CSS, PostCSS, and autoprefixer:

```sh
npm install tailwindcss@latest postcss@latest autoprefixer@latest
```

## Configuring Tailwind CSS

After installing the necessary packages, generate the Tailwind CSS configuration file:

```sh
npx tailwindcss init -p
```

This command creates a `tailwind.config.js` file at the root of your project and a `postcss.config.js` file. The `-p` flag includes PostCSS configuration. In the `tailwind.config.js` file, replace the `purge` array with the following to remove unused styles in production:

```js
purge: [
    "./pages/**/*.{ts,tsx}",
    "./components/**/*.{ts,tsx}",
    "./app/**/*.{ts,tsx}",
    "./src/**/*.{ts,tsx}"
],
```

## Including Tailwind CSS in your Application

Finally, create a new CSS file called `globals.css` in the `styles` directory and import Tailwind's `base`, `components`, and `utilities` styles:

```css
@import "tailwindcss/base";
@import "tailwindcss/components";
@import "tailwindcss/utilities";
```

Now, your Next.js application is set up with Tailwind CSS. You can start your development server by running `npm run dev` and navigate to `http://localhost:3000` to see your application in action.

## Conclusion

You've now set up a Next.js project with Tailwind CSS. This setup allows you to start building a modern, stylish web application using React and utility-first CSS. Happy coding!
