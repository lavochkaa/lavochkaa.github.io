---
title: "GitHub Galaxy: an account as a spiral"
categories: [Projects, WebGL]
tags: [threejs, webgl, github, python, visualization]
image:
  path: /assets/img/github-galaxy/overview.jpg
  alt: A spiral galaxy built from the public repositories in a GitHub account
---

A repository list tells me what I have built. I wanted a way to see how that work grew over time, so I made [GitHub Galaxy](https://github.com/lavochkaa/githubGalaxy): a navigable spiral generated from a GitHub account.

Each repository becomes an arm and each commit becomes a star. Newer commits sit farther from the core. Click an arm and the camera moves into its history: branches appear as parallel strands, merges glow brighter, and GitHub Actions runs pulse around the commits they built.

![A GitHub account turns into a galaxy; the camera moves from the overview to a repository and its commits](/assets/img/github-galaxy/demo.gif)

## Inside the scene

The client uses Three.js r186, with custom WebGL shaders for roughly 25,000 stars, gas filaments, dust lanes, the warm core, and the stellar halo. A five-level HDR bloom chain gives the brighter commits and core their glow. The spiral arm calculation is shared between JavaScript and GLSL, so what you hover matches what the GPU draws.

The selected repository has a second view for its commits, branches, pull requests, and Actions runs. The list and the stars stay in sync: selecting a commit in either place highlights it in both.

![Repository drill-down showing an arm, commit stars, and the detail panel](/assets/img/github-galaxy/repository.jpg)

## Where the data comes from

The [live demo](https://lavochkaa.github.io/githubGalaxy/) is a static snapshot of my public repositories, rebuilt on GitHub Pages every hour. It is read-only. For your own account, the local Python server fetches GitHub data at runtime; it works with the public REST API without a token, or GraphQL with a read-only token. The token stays on the server and never reaches the browser.

There is no package installation or JavaScript build step. With Python 3.10+ and a WebGL 2 browser:

```sh
git clone https://github.com/lavochkaa/githubGalaxy.git
cd githubGalaxy
python3 server.py --owner <your-github-login>
```

Open `http://127.0.0.1:8765/`. The [source and setup notes](https://github.com/lavochkaa/githubGalaxy) cover the token option and controls.

![Full galaxy overview with repository labels around the spiral](/assets/img/github-galaxy/overview.jpg)

I am interested in whether the arm picking and commit cues stay clear on different displays. If you try it, tell me what is hard to read or navigate.
