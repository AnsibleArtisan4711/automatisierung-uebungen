# Quellenverzeichnis

> **Hinweis:** Teil des Skripts „Git und Versionskontrolle“, aus der LaTeX-Quelle in Markdown umgewandelt. Die Kapitel zu den Modulen stehen als `skript.md` in den Modulordnern.

## Tatsächlich abgerufen (Stand 28.10.2026)

- Git-Dokumentation: Planned Git 3.0 Breaking Changes. <https://git-scm.com/docs/BreakingChanges.html> (Standardzweig `main`, SHA-256, reftable, `safe.bareRepository`, Fortbestand von `git checkout`). Verwendet in Modul 01 und 03.

- Git for Windows, Installationsseite: <https://git-scm.com/install/windows> (Version 2.56.0, veröffentlicht am 26.10.2026). Verwendet in Modul 01.

- Git-Dokumentation: gitignore. <https://git-scm.com/docs/gitignore> (Mustersyntax, Rangfolge der Quellen). Verwendet in Modul 02.

- Git-Dokumentation: git-merge. <https://git-scm.com/docs/git-merge> (Fast-forward, `--no-ff`, `--squash`, Konfliktmarker, `zdiff3`, Strategie `ort`). Verwendet in Modul 03.

- Git-Dokumentation: git-rebase. <https://git-scm.com/docs/git-rebase> (Ablauf, `--onto`, `--continue`/`--abort`, Warnung zu veröffentlichter Historie). Verwendet in Modul 03.

- Git-Dokumentation: git-cherry-pick. <https://git-scm.com/docs/git-cherry-pick> (`-x`, `-n`, `-m`, Konfliktbehandlung). Verwendet in Modul 03.

- Git-Dokumentation: git-push. <https://git-scm.com/docs/git-push> (Refspecs, `--force-with-lease`, `--delete`). Verwendet in Modul 04.

- Git-Dokumentation: git-pull. <https://git-scm.com/docs/git-pull> (`--ff-only`, `--rebase`, `--autostash`). Verwendet in Modul 04.

- Git-Dokumentation: git-fetch. <https://git-scm.com/docs/git-fetch> (`--prune`, Refspecs, `--depth`). Verwendet in Modul 04 und 05.

- Git-Dokumentation: git-reset. <https://git-scm.com/docs/git-reset> (Modi `--soft`, `--mixed`, `--hard`, `--keep`, `ORIG_HEAD`). Verwendet in Modul 06.

- Git-Dokumentation: git-reflog. <https://git-scm.com/docs/git-reflog> (Verfallsfristen, `HEAD@{n}`). Verwendet in Modul 06.

- Git-Dokumentation: git-bisect. <https://git-scm.com/docs/git-bisect> (`bisect run`, Rückgabewerte, `old`/`new`). Verwendet in Modul 06.

- Git-Dokumentation: git-revert. <https://git-scm.com/docs/git-revert> (`-m`, `--no-commit`). Verwendet in Modul 06.

- Git-Dokumentation: git-stash. <https://git-scm.com/docs/git-stash> (`push`, `-u`, `pop` bei Konflikten). Verwendet in Modul 06.

- Git-Dokumentation: git-blame. <https://git-scm.com/docs/git-blame> (`-L`, `-w`, `-M`, `-C`, `--ignore-rev`, `blame.ignoreRevsFile`). Verwendet in Modul 06.

- Git-Dokumentation: git-worktree. <https://git-scm.com/docs/git-worktree> (ein Branch je Arbeitsverzeichnis). Verwendet in Modul 06.

- Git-Dokumentation: git-sparse-checkout (Design-Dokument). <https://git-scm.com/docs/git-sparse-checkout> (Cone-Modus). Verwendet in Modul 05.

- Unleash: Trunk-Based Development. <https://www.getunleash.io/trunk-based-development> (Prinzipien, Feature Flags). Verwendet in Modul 05.

- GitLab-Dokumentation: Git Large File Storage. <https://docs.gitlab.com/topics/git/lfs> (Funktionsweise von LFS). Verwendet in Modul 05.

- GitHub-Dokumentation: Available rules for rulesets. <https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/available-rules-for-rulesets> (Regeltypen für Branches). Verwendet in Modul 05.

- GitHub-Dokumentation: About code owners (Version Enterprise Server 3.20). <https://docs.github.com/en/enterprise-server@3.20/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners> (CODEOWNERS-Syntax). Verwendet in Modul 04 und 05.

- Git-Dokumentation: git-tag. <https://git-scm.com/docs/git-tag> (annotierte und signierte Tags, Verifikation). Verwendet in Modul 07.

- Git-Dokumentation: git-describe. <https://git-scm.com/docs/git-describe> (`--abbrev`, `--match`, Fehlermeldung ohne Tags). Verwendet in Modul 07, 09 und 10.

- Semantic Versioning 2.0.0. <https://semver.org/> (Regeln für Haupt-, Neben- und Patch-Version, Vorabversionen). Verwendet in Modul 07.

- Keep a Changelog 1.1.0. <https://keepachangelog.com/en/1.1.0/> (Aufbau eines Changelogs). Verwendet in Modul 07 und 09.

- Git-Dokumentation: githooks. <https://git-scm.com/docs/githooks> (clientseitige und serverseitige Hooks, `pre-receive`, `--no-verify`). Verwendet in Modul 08 und 09.

- Git-Dokumentation: git-gc, git-maintenance, git-fsck, git-count-objects. <https://git-scm.com/docs/git-gc>, <https://git-scm.com/docs/git-maintenance>, <https://git-scm.com/docs/git-fsck>, <https://git-scm.com/docs/git-count-objects> (Pflege und Prüfung des Objektspeichers). Verwendet in Modul 07 und 10.

- Git-Dokumentation: git-submodule und git-clone. <https://git-scm.com/docs/git-submodule>, <https://git-scm.com/docs/git-clone> (Submodule, `--depth`, `--filter`, `--mirror`). Verwendet in Modul 07, 08 und 10.

- GitHub-Dokumentation: About commit signature verification; Removing sensitive data from a repository; About push protection; Adding a security policy (docs.github.com). Verwendet in Modul 08.

- GitLab-Dokumentation: SSH signed commits. <https://docs.gitlab.com/ee/user/project/repository/ssh_signed_commits/>. Verwendet in Modul 08.

- Git-Konfigurationsdokumentation zu `gpg.*`. <https://code.googlesource.com/git/+/master/Documentation/config/gpg.adoc>. Verwendet in Modul 08.

- git-filter-repo, README. Verwendet in Modul 08 (`--replace-text`).

- GitHub-Dokumentation: Workflow syntax for GitHub Actions. <https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax> (Trigger, Filter, `concurrency`, `permissions`; die Seite wurde nur teilweise gelesen, die ersten 100 000 Zeichen). Verwendet in Modul 08 und 09.

- Datadog, Regel „cicd-github-unpinned-actions-full-length-commit-sha“ (Drittanbieter). Verwendet in Modul 08.

- oneuptime, Blogbeitrag zu GitLab-CI-Regeln (Drittanbieter, `rules:if`/`rules:changes`). Verwendet in Modul 09; nur als Hinweis, nicht als amtliche Dokumentation.

- Legiscope, DSGVO Art. 17 (Sekundärquelle, nicht der amtliche Text; keine Rechtsberatung). Verwendet in Modul 08.

- GitHub-Dokumentation: About organizations. <https://docs.github.com/en/organizations/managing-organization-settings/about-organizations>. Verwendet in Modul 11.

- GitHub-Dokumentation: About rulesets (Enterprise Cloud). <https://docs.github.com/en/enterprise-cloud@latest/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/about-rulesets>. Verwendet in Modul 11.

- GitHub-Dokumentation: About SCIM for Enterprise Managed Users und Configuring SCIM provisioning for Enterprise Managed Users (docs.github.com, `enterprise-cloud@latest`). Verwendet in Modul 11.

- GitLab-Dokumentation: Protected branches, Branch rules, Protect your repository. <https://docs.gitlab.com/ee/user/project/repository/branches/protected.html>, <https://docs.gitlab.com/user/project/repository/branches/branch_rules/>, <https://docs.gitlab.com/user/project/repository/protect/>. Verwendet in Modul 11.

- Microsoft Learn: Branch policies and settings. <https://learn.microsoft.com/en-us/azure/devops/repos/git/branch-policies?view=azure-devops> sowie <https://learn.microsoft.com/vsts/git/branch-policies>. Verwendet in Modul 11.

- Jenkins-Dokumentation: Multibranch Pipeline. <https://www.jenkins.io/doc/book/pipeline/multibranch/>. Verwendet in Modul 11.

- oneuptime, Blogbeitrag zu Jenkins-Pipeline-Triggern (Drittanbieter). <https://oneuptime.com/blog/post/2026-01-30-jenkins-pipeline-triggers/markdown>. Verwendet in Modul 11; nur als Hinweis, nicht als amtliche Dokumentation.

- IETF, RFC 7643 (SCIM Core Schema). <https://datatracker.ietf.org/doc/html/rfc7643>. Verwendet in Modul 11.

- Git-Dokumentation: git-svn, git-bundle, partial-clone. <https://git-scm.com/docs/git-svn>, <https://git-scm.com/docs/git-bundle>, <https://git-scm.com/docs/partial-clone>. Verwendet in Modul 12.

- Microsoft Learn: Migrate from TFVC to Git. <https://learn.microsoft.com/en-us/devops/develop/git/migrate-from-tfvc-to-git>. Verwendet in Modul 12.

- GitHub-Dokumentation: Backing up a repository und GitHub Actions billing (`enterprise-cloud@latest`). <https://docs.github.com/en/enterprise-cloud@latest/repositories/archiving-a-github-repository/backing-up-a-repository>, <https://docs.github.com/en/enterprise-cloud@latest/billing/concepts/product-billing/github-actions>. Verwendet in Modul 12; Preise und Kontingente ändern sich und sind vor Entscheidungen aktuell zu prüfen.

## Weiterführend (nicht abgerufen, nicht als Beleg verwendet)

- Conventional Commits, Spezifikation 1.0.0. <https://www.conventionalcommits.org/> (in Modul 02 aus dem Wissensstand dargestellt, siehe dortigen Hinweis).

- Pro Git (Scott Chacon, Ben Straub), frei online. <https://git-scm.com/book>

- Vincent Driessen, „A successful Git branching model“ (GitFlow) und sein späterer Hinweis zur Eignung für kontinuierliche Auslieferung (Modul 05).

- Dokumentation von GitLab (Protected Branches) und Azure DevOps (Branch Policies) (Modul 05).

- Plattform-Referenzen `refs/pull/N/head` (GitHub) und `refs/merge-requests/N/head` (GitLab) (Modul 04).

- Git-Howto „Revert a faulty merge“ in der Git-Dokumentation (Modul 06).

- Git LFS, Dokumentation und Plattformgrenzen (Quotas, Preise); gitcredentials; git-lfs migrate; git-verify-commit; gitleaks; Dependabot und Renovate; GitHub „Secure use reference“ für Actions; Conventional Commits; GitHub Releases; CVSS; DSGVO Art. 5 und 17 im Originaltext (Module 07 bis 10). Diese Quellen wurden nicht abgerufen; entsprechende Aussagen beruhen auf dem Wissensstand und sind im Text als solche gekennzeichnet oder durch lokale Tests (Git 2.43, Git LFS 3.4.1) belegt.

### Hinweis.

Die Git-Dokumentation ändert sich mit jeder Version. Prüfen Sie Optionen und Voreinstellungen mit `git help <befehl>` gegen Ihre installierte Version. Rechtliche Hinweise in diesem Skript stellen keine Rechtsberatung dar.
