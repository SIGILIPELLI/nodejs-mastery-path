---
description: "Install Node.js the maintainable way: LTS vs Current releases, nvm and fnm version managers, .nvmrc files, and pinning versions per project."
---

# 02 · Installing Node & Managing Versions

Installing Node is easy. Installing it so that next year's projects, last year's
projects, your CI server, and your production container all agree on a version takes
a little more thought. This lesson sets that up once.

## Release lines: Current vs LTS

Node publishes a new **major** version roughly every six months (April and October).

- **Odd-numbered** majors (e.g. 23, 25) are *Current* only. They get new features first
  and are then abandoned after a few months.
- **Even-numbered** majors (e.g. 22, 24) start as Current and are promoted to
  **Long-Term Support (LTS)** in October. An LTS line receives bug and security fixes
  for about 30 months.

For servers and anything you deploy, use an **Active LTS** or **Maintenance LTS**
version. The official schedule lives at
[github.com/nodejs/release](https://github.com/nodejs/release); check it rather than
trusting a blog post, since the table moves every six months.

You can ask a running Node whether it is an LTS build:

```bash
node -p "process.release.lts"
```

This prints the LTS codename (for example a name like `Jod`) on LTS builds and
`undefined` on Current builds.

## Option 1: the official installer

Download from [nodejs.org](https://nodejs.org/) and run it. You get `node`, `npm`, and
`npx`. This is fine if you only ever need one version, but upgrading means reinstalling,
and global packages may need `sudo` on macOS/Linux — a common source of permission
problems.

## Option 2 (recommended): a version manager

A version manager installs Node versions into your home directory and switches between
them per shell or per project. No `sudo`, no conflicts.

=== "nvm (macOS/Linux)"

    ```bash
    # Install nvm (see github.com/nvm-sh/nvm for the current install line)
    # then, in a new terminal:
    nvm install --lts          # latest LTS
    nvm install 22             # a specific major
    nvm use 22                 # switch this shell
    nvm alias default lts/*    # what new shells get
    nvm ls                     # installed versions
    ```

=== "fnm (macOS/Linux/Windows)"

    ```bash
    # fnm is a fast Rust-based manager; install via Homebrew, winget, or its script
    fnm install --lts
    fnm use 22
    fnm default 22
    # Add to your shell profile so it auto-switches on cd:
    # eval "$(fnm env --use-on-cd)"
    ```

=== "Windows"

    Use **fnm** (above) or **nvm-windows**, a separate project with similar commands
    (`nvm install lts`, `nvm use <version>`). Note that nvm-windows is not the same
    tool as nvm and does not read `.nvmrc` automatically.

Other choices exist — Volta, asdf/mise, and your OS package manager. They all solve the
same problem; pick one and use it consistently.

## Pinning the version per project

Put a `.nvmrc` file at the project root containing just the major version:

```text title=".nvmrc"
22
```

Now `nvm use` (or `fnm use`) with no arguments picks it up, and many CI systems (for
example `actions/setup-node` with `node-version-file: .nvmrc`) read the same file.

Also declare the supported range in `package.json`:

```json title="package.json (excerpt)"
{
  "engines": {
    "node": ">=22"
  }
}
```

`engines` is advisory by default: npm prints a warning on install if the running Node
does not match. Add `engine-strict=true` to the project's `.npmrc` to turn that into an
error.

## Worked example: a fresh project with a pinned version

```bash
mkdir hello-node && cd hello-node
nvm install 22 && nvm use 22
echo "22" > .nvmrc
npm init -y
npm pkg set type=module
npm pkg set engines.node=">=22"
echo "console.log('Node', process.version, 'on', process.platform, process.arch);" > index.js
node index.js
```

The last line prints your exact Node version, OS, and CPU architecture. Commit
`.nvmrc` and `package.json`; a teammate who clones the repo runs `nvm use` and gets
the same major.

## The REPL and quick one-liners

Running `node` with no arguments opens the REPL. Useful tricks:

- `.help` lists commands; `.load file.js` runs a file in the session; `.exit` quits.
- `_` holds the last result.
- Top-level `await` works in the REPL.
- `node -e "code"` evaluates and exits; `node -p "expr"` evaluates and prints.
- `node --watch app.js` restarts on file changes — built in, no nodemon needed.
- `node --env-file=.env app.js` loads environment variables from a file (lesson 09).

## How It Actually Works

A version manager is mostly **PATH manipulation**. nvm installs each version into its
own directory, such as `~/.nvm/versions/node/v22.x.y/bin/`, containing `node`, `npm`,
and `npx`. `nvm use 22` is a shell function that rewrites your `PATH` so that
directory comes first. When you type `node`, the shell searches `PATH` left to right and
finds that version's binary. That's why nvm must be loaded in your shell profile, and
why it only affects the current shell session.

fnm and Volta take a slightly different approach — a shim or a per-shell symlink
directory — but the effect is the same: the name `node` resolves to a chosen binary.

Global packages (`npm install -g`) are installed *per Node version* under that version's
directory. Switching versions therefore "loses" your globals; this is by design, since a
native global package compiled against one Node version may not load in another. Prefer
`npx some-tool` or project-local dev dependencies over globals.

Native addons (packages with C/C++ code, like some database drivers) are compiled
against a specific **Node ABI version** (`process.versions.modules`). After switching
majors you may see "was compiled against a different Node.js version"; fix it with
`npm rebuild` or by reinstalling `node_modules`.

## Common mistakes

- **Using `sudo npm install -g`.** It leaves root-owned files in your home directory and
  breaks later installs. Use a version manager instead.
- **Mixing a system Node and nvm Node.** `which node` tells you which one you are
  actually running. If it's `/usr/local/bin/node` while you expect nvm, your profile is
  not loading nvm.
- **Pinning an odd-numbered release for production.** It goes end-of-life quickly.
- **Forgetting CI.** If CI installs "latest" while you develop on 22, you will
  eventually ship something that only works on one of them. Read the version from
  `.nvmrc` in CI too.
- **Committing `node_modules`.** It's platform-specific and huge. Commit the lockfile
  instead (lesson 04).

## Exercise

1. Install a version manager and two Node majors. Switch between them and confirm with
   `node -v` and `which node`.
2. Create a project with `.nvmrc` and `engines`. Set `engine-strict=true` in `.npmrc`,
   switch to an older major that violates `engines`, run `npm install`, and read the
   error.
3. Find your Node release line on the official release schedule and write down its
   end-of-life date.
