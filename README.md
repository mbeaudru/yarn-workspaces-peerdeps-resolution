# Working between yarn workspaces and other projects with peerDependencies

This repository is a meant to reproduce an issue we encountered in trying to work between a workspace and other repositories.

To be concrete, we have workspaces for components (more or less smart) and tooling separated in multiple packages, but "projects" are separate repositories and depends on packages from the workspace.

> **Note :** We have created a npm package "log-version" that only *console.log* its current package version to demonstrate the  issue.

## The issue

We want to be able to dev on *package-a* while it is linked in *project-a* and always have the same results than in prod, when packages are taken from the registry for instance.

- *package-a* belongs to the workspace and has a **peerDependency** on **log-version ^1.0.0**
- *package-b* and *package-c* belongs to the workspace and both have a dependency on **log-version@1.0.0**

> This makes log-version@1.0.0 be hoisted by the workspace

- *project-a* has a **dependency** on **log-version@1.1.0** and on *package-a* (version doesn't matter)

```
workspaces
  |-- node_modules
    |-- log-version@1.0.0
  |-- packages
    |-- package-a (peerDependency: log-version ^1.0.0)
    |-- package-b (dependency: log-version@1.0.0)
    |-- package-c (dependency: log-version@1.0.0)

project-a
  |-- node_modules
    |-- log-version@1.1.0
    |-- package-a
  package.json
    * dependency:
      - log-version@1.1.0
      - package-a
```

### In dev

We link *package-a* in *project-a* and run *project-a*:

- *project-a* has **log-version@1.1.0** in dependency so uses version **1.1.0** when running
- *package-a* has **log-version ^1.0.0** in *peerDependency* so running in *project-a*, should use **log-version@1.1.0**

**BUT**, *package-a* in fact uses **log-version@1.0.0** because it is found in the workspaces root *node_modules* before!

Follow the steps, having in mind that *package-a* is linked into *project-a*:

```
workspaces
  |-- node_modules // STEP 3
    |-- log-version@1.0.0 // STEP 4
  |-- packages
    |-- package-a (peerDependency: log-version ^1.0.0) // STEP 2
    |-- package-b (dependency: log-version@1.0.0)
    |-- package-c (dependency: log-version@1.0.0)

project-a
  |-- node_modules
    |-- log-version@1.1.0
    |-- package-a // STEP 1
  package.json
    * dependency:
      - log-version@1.1.0
      - package-a
```

### In production

- *project-a* has **log-version@1.1.0** in dependency so uses version **1.1.0** when running
- *package-a* has **log-version ^1.0.0** in *peerDependency* so running in *project-a*, uses **log-version@1.1.0**

When not linked, we do have:

```
project-a
  |-- node_modules
    |-- log-version@1.1.0 // STEP 2
    |-- package-a // STEP 1
  package.json
    * dependency:
      - log-version@1.1.0
      - package-a
```

## Steps to reproduce

- clone the repository
- at repository root, run `yarn prepare` to install all dependencies
- at repository root, run `yarn start` to run *project-a*

Observe that despite being in peerDependency, log-version is used both @1.1.0 and @1.0.0

## Solutions tried

### no-hoist all workspaces

Solves the problem since there's no hoisting in the workspaces root node_modules, but for medium / large projects this is not an option because it doesn't scale (huge dependencies tree) and defeats the purposes of the workspaces in the first place.

### Copy "package-a" in "project-a" using yalc

If we copy directly *package-a* in *project-a*:

- *project-a* is responsible of *package-a* dependencies install

This ensures that there will be no differences between prod & dev results.

But copy is not free and costs more horse power, because in dev mode we want files to be copied on file change, which requires to have watchers on *package-a* that will trigger the copy operation. It doesn't scale if you want to chain package links, which can happen sometimes.

### Colocate a devDep for each peerDep

We could make a trick in adding to *package-a* a devDependency for log-version and make this match the version provided by *project-a*.

Follow the steps, having in mind that *package-a* is linked into *project-a*: 

```
workspaces
  |-- node_modules
    |-- log-version@1.0.0
  |-- packages
    |-- package-a
    |-- node_modules // STEP 2
      |-- log-version@1.1.0 // STEP 3
    package.json
      * peerDependency:
        - log-version ^1.0.0
      * devDependency:
        - log-version@1.1.0

project-a
  |-- node_modules
    |-- log-version@1.1.0
    |-- package-a // STEP 1
  package.json
    * dependency:
      - log-version@1.1.0
      - package-a
```

But it requires you to make sure the project and the devDependency are the same for each peerDependency for this to work, which is a pain for the teams.

Last but not least, using a peerDependency is usually involved because you want to use a single instance of a given dependency running in your application, so this "trick" is still not an option for some cases. It still is not the same as the *unlink* workflow.

### Yarn PnP

Since plug'n play changes the way dependencies are resolved, it might entirely solve this problem.

In fact, the branch *pnp* of this repo proves that it indeed works!

But when we tried to move to PnP we encountered other issues (that we might decide to overcome anyway), so at the moment we would want things to work without PnP unless we are sure it's the only way to make things work.