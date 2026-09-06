+++
title = 'Rebuilding this website with Hugo'
date = '2026-09-06'
draft = false
description = 'Why I chose Hugo for this site, the decisions behind its shape, and how the GitHub Actions pipeline takes it from a commit to a deployed website.'
tags = ['hugo', 'web development', 'GitHub Actions']
+++

Welcome to my first post on my new website and blog. It's going to be a collection of notes about software architecture, engineering, AI, data, and the things I am learning along the way.

My old websites (a blog and photo website) were built with languages (PHP) and ecosystems/frameworks I haven't used or touched for more than 15 years. To modernize them would take a lot of time and making them secure even more time. And I since I was paying an increasing amount each year to host the sites, I decided it was time for a change!

<!--more-->

I checked the old websites to see how relevant they still were and I came to the conclusion there was not much value (outdated techniques, old photos) in them anymore. Also, the old websites had dynamic pages supported by databases and admin panels just for me. They exposed a big attack surface, and I don’t want spend the time to keep the sites up to date. It’s not just an update of a few dependencies, it's also keeping track of security issues in a ecosystem I’m no longer active in.

So, I decided to start clean. Having seen the rise of static websites (and the low cost of hosting them) I decided to give static site generation a try. That made the first technical decision important: I don’t need a big heavy feature-rich platform. I just wanted a writing tool that could turn a handful of source files into a website. [Hugo](https://gohugo.io/) was a good fit for that constraint.

## Why Hugo

Hugo is a static site generator. I write content in Markdown, describe the site structure with templates, and Hugo produces the HTML, CSS, RSS feeds, sitemap, and robots file that are published to the web. There is no application server and no database involved when someone visits the site, just a file transfer basicly. With Hugo written in Go it’s superfast, which makes posting a new blog easy and fast.

I like the advantages of a static website with Hugo:

- **The content is portable.** Posts are plain Markdown files in the repository. They are easy to review, edit, search, and move. Moving to another host would be easy: just some changes to the deployment pipeline instead of making database backups, restoring them, testing database connections and so on.
- **The runtime is boring.** A visitor receives files rather than waiting for a server-side render or an API call. This means less infrastructure to operate and less that can fail.
- **The build is quick.** A new post or a change to the website is easily deployed. No need to compile anything or complicated release process.
- **The output is easy to inspect.** The thing tested locally or in CI is the same generated output that is uploaded to GitHub Pages. This gives me the opportunity to validate my changes quick as the feedback loop is short.
- **Hosting is cheap.** In my current setup it's even free: I use GitHub Pages to host the pages. Ok, I pay a little bit per year for the domain names.

I considered the alternatives to Hugo, but the decision was less about finding the most capable framework and more about avoiding capabilities I don’t need. This site is content-led, so a static generator keeps the architecture close to the problem. Alternatives were build on Node.JS (often with high memory, making them slower) or more complex to setup. To install Hugo on Windows:

```powershell
winget install Hugo.Hugo.Extended
```

## Other decisions to make

Choosing Hugo was only the start. I needed to make a few other decisions to make the result feel like a coherent site rather than a collection of generated pages.

### Using Gen-AI

Nowadays, I use GitHub Copilot for much of my coding work and I also used it while creating this new website. For the writing, I want the ideas and first draft to remain mine. I forces me to think and to keep track of my own blog... I use AI mainly as an editor: to challenge ideas, spot gaps, and improve the final text.

### Performance and accessibility

Static websites have a good foundation for performance because there is no server-side rendering at request time. However, you can still build a website that is slow to load or render. To check the performance of the website, I decided to include [Lighthouse](https://developer.chrome.com/docs/lighthouse/) in the build pipeline. This tool can, for example, test how quickly the website is rendered in constrained environments such as older phones or on slow connections. The first tests with Lighthouse gave the website already a good score, but there was one area I could improve the website even more, which led to the following decision.

### Typography and assets are local

The typefaces are stored in the repository and served from the website. That avoids a third-party font request and means the privacy and availability characteristics of the page do not depend on an external font provider (goodbye [Google Fonts](https://fonts.google.com/)). Another advantage is that I can choose exactly which font weights are loaded, making the website even faster to load.

The same principle applies to the rest of the site: the published page should contain what it needs instead of assembling itself from a collection of runtime services. I don't need analytics, tracking scripts, or social media widgets.

### Templates

I kept the templates simple, which makes it easier to understand and maintain them. The layout is split around a few responsibilities: a base document shell, shared header and footer partials, a homepage, a section list for posts, and a single-post layout. That's all I need for the functionality I currently have planned.

The site also avoids pulling in a front-end framework like Angular or React. I thought about this, trying a new framework like React is always tempting but I decided against it. The interaction on the website is mostly navigation and reading, which I can handle perfectly well with HTML, CSS and basic JavaScript. Learning a new framework takes time and means more dependencies to keep up to date. A nice side effect is a smaller page size, but the bigger benefit is that the implementation remains understandable when I return to it later.

### Quality is part of the publishing path

A static site can, like a dynamic site, have broken links, invalid markup, inaccessible controls, or a layout that overflows on a phone. Those are build concerns, not just things I want to check by eye.

I therefore treated the generated site as a testable artifact. The checks cover HTML validity, automated accessibility, browser behavior in Chromium, Firefox, and WebKit, responsive overflow, keyboard focus, and Lighthouse accessibility and SEO scores.

In the future, I plan to add automated checks for broken links in blog posts, but that requires building some smart stuff to extract the links from the posts.

### Keeping dependencies current

A static site means fewer moving parts, but Hugo and other dependencies still need to be kept up to date. I chose Dependabot to keep the npm packages current. I also configured it to check the GitHub Actions used by the workflow, keeping the whole system up to date through PRs for me to approve. You can see my current [`.github/dependabot.yml`](https://github.com/antonrademaker/antonrademaker.github.io/blob/main/.github/dependabot.yml).

## How the GitHub pipeline works

The pipeline is defined in [`.github/workflows/site.yml`](https://github.com/antonrademaker/antonrademaker.github.io/blob/main/.github/workflows/site.yml) and runs for pull requests and pushes to `main`.

On every validation run, GitHub Actions:

1. Checks out the repository.
2. Sets up Node.js 24 and caches the npm dependencies.
3. Runs `npm ci` from `src/www`.
4. Installs the Chromium, Firefox, and WebKit browsers used by Playwright.
5. Runs `npm test`.

The `npm test` script first runs `hugo --minify`, which writes the production site to `public/`. It then validates the generated HTML, runs the Playwright and axe-core checks, and applies the Lighthouse quality gates. The tests exercise the actual generated site rather than a separate mock or test version.

If validation succeeds, the workflow uploads `src/www/public` as a GitHub Pages artifact. For a push to `main`, the deploy job then publishes that artifact to the `github-pages` environment, updating the website. Pull requests stop after validation, so they can prove that a change is ready to publish without actually deploying it.

The result is a short and straightforward path:

```text
Markdown + templates + CSS
              |
              v
      Hugo production build
              |
              v
   HTML, accessibility, browser,
      and Lighthouse checks
              |
              v
      GitHub Pages artifact
              |
              v
            Deploy
```

This setup keeps the publishing process easy to understand and maintain while giving me confidence that the generated site is in good order. There is no separate release package to prepare and no infrastructure to configure. A merged change to `main` effectively becomes the release, provided the same validation checks that protect the pull request have passed.

## What comes next

For now, this is the foundation rather than the finished website. I plan to write more about Hugo in the future, including some of the features it handles for me and the decisions behind the rest of the site.

Hugo gives both me and the site the kind of simplicity I was looking for: the content stays close to the code, the generated output is easy to understand, and the pipeline has a clear answer to the question: "What exactly are we deploying?"
