Migrate from SVN to Git
=======================

Intent
------

The aim is to migrate OpenJUMP from [Subversion](https://svn.code.sf.net/p/jump-pilot/code/) to Git while keeping the entire history of the project. This first migration attempt relies on [GitHub](https://github.com/) services but this doesn't exclude the use of other Git-repository managers such as [GitLab](https://gitlab.com/) in a near future, or even to use both based on a [repository mirroring](https://docs.gitlab.com/ee/user/project/repository/repository_mirroring.html).


Environment<a name="environment"></a>
-----------

The environment used for this migration test is:
- OS: Ubuntu 18.04
- Subversion: version 1.9.7 (r1800392)
- Git: version 2.17.1
- git-svn: version 2.17.1 (svn 1.9.7)
- Ruby: 2.5.1p57
- svn2git: Ruby tool

Git includes git-svn, a useful tool to perform all the bidirectional operations between a Subversion repository and Git. As it is possible to have Git installed without git-svn installed, please do verify that the `git svn` command can be run successfully. If not, git-svn needs to be installed.

Despite the fact that git-svn is a powerful tool, [svn2git](https://github.com/nirvdrum/svn2git) is here preferred. svn2git is a ruby wrapper around git's native SVN support through git-svn which easily allows migrating projects from Subversion to Git while keeping the trunk, branches and tags where they should be. It uses git-svn to clone an svn repository and does some clean-up to make sure branches and tags are imported in a meaningful way, and that the code checked into master ends up being what's currently in your svn trunk rather than whichever svn branch your last commit was in. Using git-svn alone, even with the options `-T trunk -b branches -t tags`, wouldn't create directly a proper Git correspondence, and additional steps would be required. Even if these extra steps could be automated (see [here](https://blog.axosoft.com/migrating-git-svn/) for instance), svn2git works well and allows us to avoid them.

Make sure these required prerequisites are installed in your local environment. If not, you can do so for a Debian-based system by using:

```sh
sudo apt update
sudo apt install subversion git git-svn ruby
sudo gem install svn2git
```

GitHub organisation and permissions
-----------------------------------

An organisation is essentially a way to put one or more repositories under a single umbrella. It regroups individuals who can be defined based on these four roles / permission levels:
- Owner — Full administrative access to the entire organisation, including the global settings and all the repositories,
- Billing manager — Mostly used if paid plans are in place, with no access to the repositories,
- Outside collaborator — Not a member of the organisation, but with rights to read / write, or with admin permissions to one or more repositories owned by the organisation,
- Member (anyone else) — Rights to see every member and non-secret team in the organisation, and to create new repositories.

A team is different from an organisation. A team is a group of organisation members. It allows a great control of the permissions, for example at a repository level.

In the context of this migration test, a GitHub organisation named `openjump-gis` has been created to put together some OJ related repositories into a single space, e.g. core, plug-ins, etc. All the active OJ developers are owners of this organisation. No team has been created yet.

Note: the organisation isn't named `openjump` as initially envisaged because it wasn't available as a username or organisation name. `openjump-gis` is thus used as a temporary name.


Migration process
-----------------

The migration process is described in the different steps below.


### 1. Create a mapping between the SVN and Git users

SVN commits are based on usernames, whereas Git requires of a full name (first and last names) and an email address. As the idea is to keep track of the entire SVN commit history, a mapping between the SVN and Git users is required. In other terms, each SVN username needs to be converted to match the required Git author information:

`svn_author_username = Firstname Lastname <email@example.com>`

Rather than using the personal email addresses of the SVN authors, the SourceForge user email addresses are used, i.e. username@users.sourceforge.net, as Ede suggested. The format is therefore:

`svn_author_username = Firstname Lastname <svn_author_username@users.sourceforge.net>`

Each author is then added into an `authors.txt` file. The different steps used to create this `authors.txt` file are described below, but for convenience, this file is already available in the [additional files folder](./additional_files/).

To retrieve a list of all Subversion committers, and to create a first mapping in the required format, the following bash command can be run in the local SVN working directory for your SVN repository:

```sh
cd svn-checkout-directory
svn log -q | awk -F '|' '/^r/ {sub("^ ", "", $2); sub(" $", "", $2); print $2" = "$2" <"$2"@users.sourceforge.net>"}' | \
    sort -u > ../authors.txt
```

or directly using a remote repository. The latter is recommended as it retrieves all the authors of the entire SVN repository, and not only the ones in a possible partial, local checkout repository:

```sh
svn log -q https://svn.code.sf.net/p/jump-pilot/code/ | \
    awk -F '|' '/^r/ {sub("^ ", "", $2); sub(" $", "", $2); print $2" = "$2" <"$2"@users.sourceforge.net>"}' | \
    sort -u > authors.txt
```

Both provide the required mapping format, i.e.:

`svn_author_name = svn_author_name <svn_author_name@users.sourceforge.net>`

Nevertheless, based on SourceForge profiles and the project mailing-list, each author has now a matching first and last names, i.e.:

`svn_author_name = Firstname Lastname <svn_author_name@users.sourceforge.net>`

For those interested in the commit statistics, it is also possible to count the number of contributions for each author, based on the following Python script:

```python
#!/usr/bin/env python

import sys
authors = {}
for line in sys.stdin:
    if line[0] != 'r':
        continue
    author = line.split('|')[1].strip()
    authors.setdefault(author, 0)
    authors[author] += 1
for author in sorted(authors):
    print(author, authors[author])
```

Save this script in a file named authorfilter.py, and use the following the following bash commands:

```sh
$ chmod +x authorfilter.py
$ svn log -q https://svn.code.sf.net/p/jump-pilot/code/ | ./authorfilter.py > authors-stats.txt
```


### 2. Convert the Subversion repository

To perform the conversion of the Subversion repository into a local Git one, svn2git is preferred to svn-git for the reasons already given in the [Environment section](#environment), mainly because it has the advantage to automatically convert the SVN tags and branches into Git tags and branches, with the appropriate Git structure.

If needed, svn2git also allows to exclude some parts of the SVN repository during this conversion process, based on exclusion patterns (see ), which could be useful to restructure some parts of the project, such as the documentation for instance, and therefore diminish the size of the main repository. Indeed, it is often advised to avoid adding binary files to a Git repository due to the way Git stores history. More information about this last point is available [here](https://docs.microsoft.com/en-us/azure/devops/learn/git/centralized-to-git#binary-files-and-tools).

Based on svn2git, the following bash commands are used to convert the OpenJUMP core SVN repository. This includes the conversion of the authors based on the `authors.txt` file previously created (this file needs to be in the same directory, here `openjump-migration`, otherwise its path needs to be specified):

```sh
mkdir openjump-migration
cd openjump-migration
svn2git svn://svn.code.sf.net/p/jump-pilot/code/core --revision 1:6242 --authors authors.txt
```

The revision numbers 1 and 6242 are respectively the revision number at which the SVN global repository has been created and the one for the 1.15 release.

Because the core repository has a standard layout (trunk, branches, tags), there is no need to use the options `--trunk`, `--branches`, `--tags`, or `--notrunk`, `--notags`, `--nobranches`. Some of these options will be required to convert the [plug-ins repository](https://svn.code.sf.net/p/jump-pilot/code/plug-ins/). For instance, the [CsvDriver plug-in](https://svn.code.sf.net/p/jump-pilot/code/plug-ins/CsvDriver/) could be converted using:

```sh
svn2git svn://svn.code.sf.net/p/jump-pilot/code/plug-ins/CsvDriver --revision 1:6242 --trunk trunk --nobranches --notags --authors authors.txt
```

This first migration attempt only took into account the commit history between revision 859 and revision 6242, as 859 (in fact, 858, as859 has been wrongly picked up the first time) is the first OpenJUMP core related revision. Therefore, the bash command used during this conversion has been:

```sh
svn2git svn://svn.code.sf.net/p/jump-pilot/code/core --revision 859:6242 --authors authors.txt
```

This takes a good five hours to process from revision 859. Thus it should probably take at least six hours to convert the SVN repository from revision 1 to the revision used during the final migration.

Finally, there is no need to convert `svn:ignore` to `.gitignore`, as there is none in the OpenJUMP core SVN repository. If needed for another repository, this `git-svn` command can be used:

```sh
git svn show-ignore > .gitignore
```

In case the `svn:ignore` of all the directories need to be converted, this command will show them all in a Git compatible format:

```sh
git svn show-ignore
```

Then, to create a matching `.gitignore` into each directory, use:

```sh
git svn create-ignore
```

**Important note:** the first conversion attempt was based on the use of the *ra_serf* repository access module provided by Subversion. This module is used for accessing a repository via WebDAV protocol using serf, and can handle both `http` and `https` schemes. So the conversion used this bash command:

```sh
svn2git https://svn.code.sf.net/p/jump-pilot/code/core --revision 859:6242 --authors authors.txt
```

This attempt failed due to the following error: `ra_serf: The server sent a truncated HTTP response body`.

A second conversion has been attempted, based on the use of the *ra_svn* Subversion module rather than the *ra_serf* module. The *ra_svn* module is used for accessing a repository using the `svn` network protocol. This second attempt succeeded.


### 3. Initialise a new Git repository on GitHub

First, you need to [log in to your GitHub account](https://github.com/login). Then, as owner of the `openjump-gis` organisation, you can create a new repository by clicking the `New` repository button available on the [organisation page](https://github.com/openjump-gis) (top-right) or by directly using this [link](https://github.com/organizations/openjump-gis/repositories/new) provides a direct access.

You have now to choose a repository name, fill in the repository description, and select the Public option, in order to allow anyone to see it.

There is an option to initialise this repository with a README file, to add a .gitignore file, or/and a licence file. Nevertheless, to make easier the initial import from an existing local repository (like in our context), it is recommended to not populate this repository. This could indeed later lead to a merging problem between the local repository and the remote one, with Git refusing to merge unrelated histories.

These files (README.md, .gitignore, and LICENSE.md) can be added later on, and for convenience, are already available in the [additional files folder](./additional_files/). The README file contains a disclaimer about the current test status of this new repository. The current .gitgnore includes a basic Java configuration as well as some specific IDE related configurations such as Eclipse and IntelliJ IDEA. It is nevertheless quite permissive with the jar, zip and tar.gz extensions because the current SVN repository contains some of these. The LICENSE file is the default GNU General Public License v2.0 provided by GitHub.

Finally, click the `Create repository` button.

At this stage, you just set up a new empty, public repository on GitHub, part of the `openjump-gis` organisation. In the context of this migration test, this repository has been named `openjump-migration` and is can be accessed from [https://github.com/openjump-gis/openjump-migration.git](https://github.com/openjump-gis/openjump-migration.git).


### 4. Push the local repository resulting from svn2git to GitHub

Based on the previous naming, you should now have an `openjump-migration` directory containing the OpenJUMP core SVN repository newly converted into a Git structure. `openjump-migration` is thus a Git repository.

In the root of this local Git repository, add locally all the files with the command:

```sh
git add .
```

Commit locally:

```sh
git commit -m "SVN to Git migration test / JTS update test: initial commit"
```

Add a new remote Git repository, using the traditional name `origin`, for your local Git repository:

```sh
git remote add origin https://github.com/openjump-gis/openjump-migration.git
```

Note that you could also use, depending on your Git preferences:

```sh
git remote add origin git@github.com:openjump-gis/openjump-migration.git
```

This second solution, based on `git@github.com:username/repositoryname`, is for use with the `SSH` protocol. More details are provided in the note below.

It is now time to push your local repository to the remote one. But before doing that, in case you are new to Git or/and GitHub, you need to configure Git locally.


**Local git configuration**

You need to set your account's default identity. This can be done globally or only for a single repository. Globally first:

```sh
git config --global user.email "email@example.com"
git config --global user.name "Your Name"
```

Or locally, by omitting the --global option, and thus to set the identity only in this repository:

```sh
git config user.email "email@example.com"
git config user.name "Your Name"
```

**Note:** if you enable two-factor authentication (2FA) on your GitHub account, you are unable to push your changes via HTTPS. To do so, you need to [create a Personal Access Token (PAT)](https://docs.github.com/en/github/authenticating-to-github/creating-a-personal-access-token). Use this access token as your password. To avoid providing your credentials every time, you can use the [Git credential cache](https://git-scm.com/docs/git-credential-cache) or, but less secure, the [Git credential storage](https://git-scm.com/docs/git-credential-store). Another alternative consists in using the [SSH protocol to connect and authenticate to GitHub](https://docs.github.com/en/github/authenticating-to-github/connecting-to-github-with-ssh). With SSH keys, you can connect to GitHub without supplying your username or password (or PAT) at each visit.


**Back to the future**

You can now push the master branch of your local Git repository to GitHub:

```sh
git push -u origin master
```

At this point, the branches and the tags defined in the initial SVN repository have not been yet created in the remote repository. To push all the local branches into the remote repository:

```sh
git push --all origin
```

And to push all the local tags into the remote repository:

```sh
git push --tags origin
```


### 5. Push some additional files

You can now add the additional README.md, .gitignore, and LICENSE.md files. Just copy these from the [additional files folder](./additional_files/) into your local Git repository, and push them to the server. Individually:

```sh
git add README.md
git commit -m "Add initial README"
git add .gitignore
git commit -m "Add .gitignore"
git add LICENSE.md
git commit -m "Add initial LICENSE"
git push -u origin master
```

or using a single commit:
```sh
git add .
git commit -m "Add initial README, .gitignore, and initial LICENSE"
git push -u origin master
```

At this point, you should have a clone of the OpenJUMP core SVN repository converted into a Git structure, and hosted on GitHub under the umbrella of the `openjump-gis` organisation. Hope so!
