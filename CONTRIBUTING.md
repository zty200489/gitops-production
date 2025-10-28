## How to name and manage branches

This repository follows the following rules when managing different braches as described in this [issue](https://gitlab.lan/gitops/production/-/issues/3):

- The `main` branch is persistent and serves as the single source of truth for fluxcd to fetch and reconcile accordingly. No commits should ever be directly committed or pushed to the `main` branch to keep the `main` branch clean and manageable.

- The `dev/*` branch is ephemeral and serves as the pre-merging branch for staging and collecting any changes to be merged to the `main` branch. This prevents unfinished configurations from contaminating the `main` branch and breaking fluxcd reconciles. Each `dev/*` branch is checked out from the `main` branch independently and ceases to exist when merged into the master branch. This allows for squash merging to better clarify and rephrase multiple commits into one meaningful development commit.

- The `adjust` and `hotfix` branches are ephemeral branches checked out from the `main` branch for temporary scaling and/or emergency hotfixes. They need to be merged into both the `main` branch and any related `dev/*` branch before meeting the end of its lifecycle, and are deleted. Adjustment and hotfixes are generally small changes and important; therefore, we do not squash-merge anything from the `adjust` and `hotfix` branch.

## How to write commit messages:

Commit messages should start with the type of this commit in one of the following: `hotfix`, `adjust`, and `dev`, in with the branch this commit is in. Followed by a comma and the actual short commit message. The commit message need NOT begin with capital letters and should be concise with what exactly is done in this commit.
