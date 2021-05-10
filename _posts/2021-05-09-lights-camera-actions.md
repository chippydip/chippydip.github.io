---
layout: post
title:  "Lights, Camera, Actions!"
---

I started the week off setting the groundwork for a new React-based app. The code I have from four years ago should technically still function, but a lot has changed over that time. Since there wasn't much too the app to begin with, I decided to create a fresh `main` branch as a virgin slate. Step one was to [Create React App](https://create-react-app.dev/) to bootstrap the application and setup some best practices. This should hopefully make it easier to manage the myriad of node dependencies required for modern React app development without going crazy.

Once I had that in place, I wanted to figure out how to setup CI builds through [Github Actions](https://github.com/features/actions). This is a relatively new feature I've never used before, so it took some googling to study up on how everything worked and setup [my first workflow](https://github.com/chippydip/spacegame/blob/89e668130ea685bb5c4c01f9a053f5c422deb1ff/.github/workflows/main.yaml). This is mostly boilerplate taken from several other blog suggestions online and smashed together to setup what I wanted.

The workflow starts off setting things up to trigger off of any commits to `main` or PRs that target the branch:
```yaml
name: Build & deploy

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main
```

It contains two jobs to execute, `build` and `deploy`. The build action starts off by simply checking out the code that triggered the workflow, using a standard linux VM. This VM images includes node/npm/yarn by default, so I didn't need to explicitly install them:
```yaml
    name: Build
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v2
```

The next bit took a little more googling to figure out the right way to handle thing. Installing node modules can take a decent chunk of time. Github Actions had the ability to cache certain build artifacts and restore them based on a key, we can use this to preserve the local yarn cache to speed up builds if neither `yarn.lock` nor the VM's operating system version has changed:
```yaml
    - name: Get yarn cache directory path
      id: yarn-cache-dir-path
      run: echo "::set-output name=dir::$(yarn cache dir)"

    - name: Cache node modules
      uses: actions/cache@v2
      with:
        path: ${{ steps.yarn-cache-dir-path.outputs.dir }}
        key: ${{ runner.os }}-yarn-${{ hashFiles('**/yarn.lock') }}
        restore-keys: |
          ${{ runner.os }}-yarn-
    - name: Install project dependencies
      run: yarn --frozen-lockfile --prefer-offline 
```
The first step is simply checking where the yarn cache directory actually is on this VM and storing that to a variable to use in the next step. It then tells the Github Action runner to cache the contents of that directory with the given composite key (the runner's OS and a hash of the `yarn.lock` file). There's also a fallback `restore-keys` parameter saying if we can't find an exact match, to just use anything that was cached on the same OS since presumably it should have at least some of the target packages already. Finally, it does the actual `yarn install`. By default `yarn` will check the network for package updates when installing anyway, so we use the `--prefer-offline` flag to tell it to skip the network check if the exact package it's looking for can be found in the local cache.

**Edit:** A reader suggested I add the `--frozen-lockfile` flag to the `yarn` command here. I had totally forgotten about that, but it's definitely [best practice](https://classic.yarnpkg.com/en/docs/cli/install/) for ensuring reproducible builds in a CI environment:
> If you need reproducible dependencies, which is usually the case with the continuous integration systems, you should pass `--frozen-lockfile` flag.

The rest of the job is fairly straightforward `yarn` commands to build, test (with coverage), upload the coverage files to [`codecov`](https://codecov.io/gh/chippydip/spacegame), and then save the build artifacts so they can be used in a follow-up job.
```yaml
    - name: Build project
      run: yarn build
    
    - name: Run tests
      run: yarn test -- --coverage

    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v1

    - name: Upload production-ready build files
      uses: actions/upload-artifact@v2
      with:
        name: production-files
        path: ./build
```

Once the build is complete (assuming there were no errors), a separate `deploy` job is triggered. This job can run on a separate build machine and will trigger only for commits to `main` (so it shouldn't run for PRs). It grabs the build files from the previous step and then simply deploys them to [Github Pages](), which is just done by checking in the built files to a special `gh-pages` branch in the repo and then updating some repo settings to tell Github to deploy from there.
```yaml
  deploy:
    name: Deploy
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
    - name: Download artifact
      uses: actions/download-artifact@v2
      with:
        name: production-files
        path: ./build

    - name: Deploy to gh-pages
      uses: peaceiris/actions-gh-pages@v3
      with:
        github_token: ${{ secrets.GITHUB_TOKEN }}
        publish_dir: ./build
```

And with that, we have CI/CD setup with the latest version of code from `main` [always available](https://chippydip.github.io/spacegame/). Taking care of the build automation up front means I get to save time whenever I make changes right from the start of the project, so it really pays to invest a little time in tooling up front on a project.

## Goodbye Example
I decided the next step should be to [replace the example React App](https://github.com/chippydip/spacegame/commit/bef95c122fb0b2e691792293f8821430a3597fe5) with the old code I had from four years ago. This was fairly straightforward, I just needed to add [Three.js](https://threejs.org/), pull over the React part of the old app, and then fix up a few APIs that had changed with the newer version of `THREE` as well as some tweaks to make the updated linter happy. Unfortunately this did cause a [problem with unit tests](https://github.com/chippydip/spacegame/actions/runs/823568054) even trying to do a mount test of the new components due to the `WebGL` and `Three.js` dependencies, so for now I've just disabled the test and will come back to it after I've done some more refactoring work.

I took a [quick pass](https://github.com/chippydip/spacegame/commit/5ad8ce0fbd38b2c8d81626fb0bb829a21239f1ac) over the app to update it to use functional-style React components. This brought to light a fundamental design issue with the old code: the solar system simulation and the presentation (drawing) of that simulation were much too lightly coupled. That got me thinking about how I should be structuring the code going forward.

## Entity Component System
[ECS](https://en.wikipedia.org/wiki/Entity_component_system) is a powerful design pattern for things like games which have a large number of object (or Entities) in the game world where each Entity has a set of Components which represent different attributes of the object. The final piece is a System which is the code that operates on a set of Entities with a specific set of Components. So a physics System might process all Entities with both a `Position` and `Velocity` component each game tick to apply forces and keep track of the motion of these objects. This is in contrast to a more object-oriented design where each thing in the world would be part of a large class inheritance tree. Each part of the tree might add behaviors to the objects and every tick the game would loop through a big list of objects and run all of the update logic for each object in turn. Perhaps a good way to think of ECS in comparison is as a component-oriented software design. Processing a given system/set of components for all objects at once before moving on to another system/components turns out to be much more performant as well as more easily extensible and simpler to reason about.

This seems like a perfect fit to solve the coupling problem I currently have with the game. The simulation system shouldn't care if an object has an extra component to track Three.js data which should improve the system quite a bit. I did some research on ECS options in JavaScript/TypeScript, played around a little with what my own version might look like, and then started to research options that could leverage `WebAssembly` (likely using `rust`). The latter seems like the best solution long-term and would be a great excuse for me to finally learn rust and WebAssembly, but since the current game is quite limited I may just throw together a quick ECS system of my own in TypeScript for now so I can keep moving forward with the game and then circle back to the WASM/rust option later as the world expands and performance becomes more important.

## Next Week
So, my plan right now is to throw together a basic ECS system and use that to decouple the rendering logic from the solar system simulator. That should finally set me up to start playing around with the render code, perhaps look to replace Three.js with my own WebGL code, or perhaps just leave it as-is and start working more on the simulation side of things. At some point I do need to add some actual gameplay afterall!
