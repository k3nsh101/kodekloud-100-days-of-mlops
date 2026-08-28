### Task

The xFusionCorp Industries ML platform team recently shipped a pull request (PR) titled **Add speculative hashing scaffold**; this PR was merged by an admin without prior review to facilitate the upcoming Wednesday release cut. Subsequently, this change has caused a lint regression, resulting in a persistent red status on the `main` branch. According to the team's rollback policy, reverts should not be executed from the command line, and force-pushes are strictly prohibited. Instead, reverts must be processed through a PR, following the same procedure used for the initial breakage. Your task is to utilize Gitea's **Revert** button on the merged PR to remove the change from the `main` branch, thereby restoring the CI status to green.

1. The Gitea UI is on port `3000` (**Gitea** button). Admin credentials: `gitea-admin` / `gitea2026`. The repo is at `http://localhost:3000/gitea-admin/fraud-detector` and a working clone is at `/root/code/fraud-detector` (on main). No local git commands are needed — the revert happens entirely in the Gitea UI.

2. The starting state: the merged PR `Add speculative hashing scaffold` (visible under **Pull Requests → Closed**) landed a regression on `main`, and the latest `CI` run on `main` (on the **Actions** tab) is red. The rollback path is Gitea's built-in revert: reverting the merged commit through a new revert branch and a second, reviewable revert PR — with its own `lint` + `test` checks — that merges the reverted state back onto `main` rather than force-pushing or hand-editing.

3. The end state must include:
   - The original **Add speculative hashing scaffold** PR is still merged (reverts do not re-open the original PR).
   - A second, separate PR with a title starting with `Revert` exists and is merged.
   - `main`'s HEAD commit message contains `Revert`.
   - `main`'s HEAD commit SHA reports combined CI status `success`.

The Revert button is the Gitea (and GitHub) equivalent of the incident-response `pager → hotfix → revert-PR → merge` playbook. It does two things the command line does not: it creates a human-reviewable PR so the rollback is audit-traceable, and it runs the full CI pipeline against the reverted state before the revert lands on main. That is the difference between fixing production and rolling production back safely.

### Solution

- Log in to Gitea

- Go to the merged PR

- Click on the merged commit

  <img src="./assets/assets_83/closed_pr.png" alt="Closed PR details" />

- Revert the commit with the option `Create new branch for this and start a pull request` selected

  ```
  Operations -> Revert
  ```

  <img src="./assets/assets_83/revert_option.png" alt="Revert option" />

- Create new PR

  <img src="./assets/assets_83/new_pr_option.png" alt="Create new branch and PR" />
    
  <br />

  <img src="./assets/assets_83/new_pr.png" alt="New PR" />

- Merge the new PR
