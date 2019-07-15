# git-move
Git branching strategy optimised for understandability

This is a work in progress. 

Its purpose is to document the Git branching / releasing strategy I've been using successfully in production for the past few years. I call it Git Move.

It is optimised for understandability and practicality. The goal is to make contributing to a project, shipping it, and seeing  what is currently shipped as well as the full history of what was shipped and when as simple as possible. 

It's inspired by Git Flow, and covers all the usecases of Git Flow (as far as I am aware), but makes changes to make the above things much easier by default, without having to make much effort and/or be an expert user of Git.

@Todo description of the branching strategy here

Here is an implementation as a `sh` script in a git alias `git move`

```
[alias]
  move = "!\
    datetime() { \
      date +%Y-%m-%d_%H-%M-%S; \
    }; \
    changelog() { \
      last_release=$1; \
      commit_to_ship=$2; \
      if [ -z "$last_release" ]; then \
        printf \"This is the first ever release.\n\nChanges to ship:\n\"; \
        git log --color --pretty=format:\"  - %h (%an) %s\" $commit_to_ship; \
      else \
        printf \"Changes to ship:\n\"; \
        git log --color --pretty=format:\"  - %h (%an) %s\" $last_release..$commit_to_ship; \
      fi; \
    }; \
    get_version() { \
      branch=$1; \
      commit_to_ship=$2; \
      if [[ \"$branch\" == \"master\" ]]; then \
        printf \"$commit_to_ship\"; \
      else \
        master_fork_commit=$(git rev-parse --short $(git merge-base --fork-point master)); \
        printf \"$master_fork_commit-$commit_to_ship\"; \
      fi; \
    }; \
    get_release_tag() { \
      branch=$1; \
      commit_to_ship=$2; \
      version=$(get_version $branch $commit_to_ship); \
      if [[ \"$branch\" == \"master\" ]]; then \
        printf "release--${version}--$(datetime)"; \
      else \
        printf "release-hotfix--${version}--$(datetime)"; \
      fi; \
    }; \
    ship() { \
      branch=$1; \
      commit_to_ship=$2; \
      last_release=$(git tag --sort=-creatordate | grep release- | head -n 2 | tail -n 1); \
      version=$(get_version $branch $commit_to_ship); \
      if [[ \"$branch\" == \"master\" ]]; then \
        printf \"\n🚚 Shipping version [$version] from master branch\n  - commit: $commit_to_ship\n  - most recent release: $last_release\n\n\"; \
        changelog $last_release $commit_to_ship; \
        read -p \"Ship?\n  ✓ 'yes' to continue\n  ✗ enter / anything else to cancel\n\n> \"; \
      else \
        printf \"\n🚚 Shipping hotfix version [$version] from $branch branch\n  - commit: $commit_to_ship\n  - most recent release: $last_release\n\n\"; \
        changelog $last_release $commit_to_ship; \
        printf \"\nℹ️ Note: This is a hotfix release, since you are shipping from the $branch branch and not from master.\n\n\"; \
        read -p \"Ship hotfix?\n  ✓ 'yes' to continue\n  ✗ enter / anything else to cancel\n\n> \"; \
      fi; \
      if [ \"$REPLY\" != \"yes\" ]; then \
        printf \"\n  - cancelled\n\n\"; \
        exit 0; \
      fi; \
      tagname=$(get_release_tag $branch $commit_to_ship); \
      printf \"tag commit $commit_to_ship with $tagname\"; \
    }; \
    args_error() { \
       printf \"\nAllowed arguments\n  1: { ship, version, tag }\n  2: Commit [optional, defaults to HEAD]\n\n\"; \
    }; \
    f() { \
      if [ \"$#\" -gt 2 ]; then \
        args_error; exit 0; \
      fi; \
      branch=$(git symbolic-ref --short HEAD); \
      commit=${2:-HEAD}; \
      commit_to_ship=$(git rev-parse --short $commit); \
      if [[ \"$1\" == \"ship\" ]]; then ship $branch $commit_to_ship; \
      elif [[ \"$1\" == \"version\" ]]; then get_version $branch $commit_to_ship; \
      elif [[ \"$1\" == \"tag\" ]]; then get_release_tag $branch $commit_to_ship; \
      else \
        args_error; exit 0; \
      fi; \
    }; \
    f"
```
