# Stacked

A command to automatically rebase stacked branches in git.

This is useful for when you have multiple branches in the review process that
are all based on each other. When you get some feedback and make some changes
in one of the base branches, this command is helpful to automatically rebase
the dependent branches onto the new base branch.

## Installation

If you're using npm: `npm install -g git-stacked`

Otherwise, paste the shell script at the bottom of this README into your .zshrc
or .bashrc file.

## Config

It's important that you set the name of the main development branch that you
initally branched from and will eventually merge your features into. The
default is `main` and you can change it like this:

    $ stacked set-main <branch name>

## Usage

 1. Make your changes anywhere in the branch you want to change and commit.
 2. Run `stacked` to automatically rebase and force push all dependent
    branches.

## Detailed description of what this actually does

It's important to understand what this command actually does so you don't
accidentally mess up your branches. It does this:

  1. For each of the historical tips of the new base-branch in reverse
     chronological order and:
  2. For each of the the child-branches that still contain the historical
     tip it:
  3. Rebases everything from after the historical tip to the end of the
     child-branch onto the new base-branch.
  4. `git push --force` after rebasing, for each child-branch.

I think git only holds a limited number of historical reflogs or only for a
limited amount of time. This should work reliably if it is run within a few
weeks of making the changes with a normal amount of git activity.

## Shell script

```
stacked() {
  if [[ "$1" == "set-main" ]]
  then
    echo "${2:-main}" > "$HOME/.stackedrc"
    echo "Updated main branch to ${2:-main}"
    return
  fi

  stacked_main_branch="$(cat "$HOME/.stackedrc" 2> /dev/null || echo main)"

  current_branch="$(git rev-parse --abbrev-ref HEAD)"

  commit_date="$(date +%s) +0000"

  # For all the historical tips of this branch
  for tip in $(git rev-list --walk-reflogs "$current_branch")
  do
    # If this tip was not part of the development branch
    if ! git merge-base --is-ancestor "$tip" "$stacked_main_branch"
    then
      # For each branch containing that tip
      for branch in $(git for-each-ref --format='%(refname)' --contains "$tip" refs/heads | sed s=^refs/heads/==)
      do
        # If these branches are not already on the same line
        if ! git merge-base --is-ancestor "$current_branch" "$branch" && ! git merge-base --is-ancestor "$branch" "$current_branch"
        then
          echo "$branch used to be a descendent of previous tip $tip, and is not currently in line with $current_branch"

          # Rebase branch onto the current branch
          # (only commits after the historical tip)
          if ! GIT_COMMITTER_DATE="$commit_date" git rebase --onto "$current_branch" "$tip" "$branch"
          then
            echo
            echo "There was a conflict when rebasing one of the child branches"
            echo "Please resolve the conflicts and finish rebasing."
            echo "After that, check out the original branch and run stacked again to continue."
            echo
            return
          fi
          git push --force 2> /dev/null
        fi
      done
    fi
  done

  git checkout "$current_branch"
}
```
