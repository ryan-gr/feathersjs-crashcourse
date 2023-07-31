# Building the App for / during Production

The listed commands to run Feathers in production mode are as follows:

```npm run compile```

```npm run start```

Let's break down what these do exactly.

> `compile` and `start` here are actually scripts defined in `package.json`. They could actually be running any command that we want - but we are going to take a look at what Feathers generates first!



## Compilation

The compile command, ```npm run compile```, has two components: `shx rm -rf lib/ && tsc`

The first calls for the deletion of the `lib/` folder, the folder that the results of the compilation is dumped into - just to ensure that there are no leftover files from previous compilations. `shx` here is a utility that allows the running of unix commands on most platforms, and is there to ensure the command works on Windows, Mac, and Linux.

`tsc` is the command to run the TypeScript compiler. The compiler will basically take the TypeScript files in the Feathers source code, and compile them to JavaScript into the `lib/` folder. Primarily, the configuration for this step is found in the `tsconfig.json` file at the project root.

> Note that the `tsconfig.json` file also contains the configuration for `ts-node`, the utility used to run the dev version of Feathers.
