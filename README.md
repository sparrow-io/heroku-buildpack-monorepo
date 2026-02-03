# Sparrow Monorepo Buildpack

Custom Heroku buildpack that **keeps the monorepo structure intact** so relative gem paths work correctly.

## The Problem

Standard monorepo buildpacks (like `timanovsky/subdir-heroku-buildpack`) move your app subdirectory to the build root, which breaks relative path references like:

```ruby
gem 'tnef', path: '../../packages/ruby/tnef'
```

Because after moving `apps/api` to root, `../../packages` would resolve to `/packages` (outside the build directory).

## The Solution

Instead of moving files, this buildpack:
1. **Keeps the entire monorepo structure intact**
2. Creates a placeholder `Gemfile` at root for Ruby buildpack detection
3. Sets `BUNDLE_GEMFILE` to point to your actual Gemfile (e.g., `apps/api/Gemfile`)
4. Generates a root `Procfile` that `cd`s into your app directory

This way, relative paths like `../../packages/ruby/tnef` resolve correctly to `/app/packages/ruby/tnef`.

## Config Vars

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `PROJECT_PATH` | No | `apps/api` | Path to the app subdirectory |

## Usage

```bash
# 1. Clear existing buildpacks
heroku buildpacks:clear -a your-app

# 2. Add this buildpack FIRST
heroku buildpacks:add https://github.com/sparrow-io/sparrow-platform.git#main:infrastructure/heroku-buildpack-monorepo -a your-app

# 3. Add Ruby buildpack
heroku buildpacks:add heroku/ruby -a your-app

# 4. Set PROJECT_PATH (optional, defaults to apps/api)
heroku config:set PROJECT_PATH=apps/api -a your-app

# 5. Deploy
git push heroku main
```

## What It Does

During build:
1. Verifies `PROJECT_PATH` exists and contains a Gemfile
2. Creates placeholder `/app/Gemfile` (for Ruby buildpack detection)
3. Creates `/app/.bundle/config` with `BUNDLE_GEMFILE: apps/api/Gemfile`
4. Creates `/app/.profile.d/monorepo.sh` to set `BUNDLE_GEMFILE` at runtime
5. Generates root `Procfile` from `apps/api/Procfile` (wrapping commands with `cd apps/api &&`)

## Directory Structure

After buildpack runs, the slug contains:
```
/app/
├── Gemfile              # Placeholder (for detection only)
├── Procfile             # Generated: "web: cd apps/api && bundle exec puma..."
├── .bundle/config       # Sets BUNDLE_GEMFILE=apps/api/Gemfile
├── .profile.d/
│   └── monorepo.sh      # Runtime BUNDLE_GEMFILE export
├── apps/
│   └── api/
│       ├── Gemfile      # Your actual Gemfile (BUNDLE_GEMFILE points here)
│       ├── Gemfile.lock
│       └── ...
└── packages/
    └── ruby/
        └── tnef/        # ../../packages/ruby/tnef resolves correctly!
```

## Verification

After deploying, verify with:
```bash
heroku run bash -a your-app

# Check BUNDLE_GEMFILE is set
echo $BUNDLE_GEMFILE
# Should output: /app/apps/api/Gemfile

# Check paths exist
ls -la /app/apps/api/Gemfile
ls -la /app/packages/ruby/tnef

# Verify bundle works
cd /app/apps/api && bundle check
```
