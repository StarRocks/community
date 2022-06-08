# StarRocks Community
[![Open Source Love](https://firstcontributions.github.io/open-source-badges/badges/open-source-v1/open-source.svg)](https://github.com/firstcontributions/open-source-badges)

Welcome to the StarRocks community! We're glad to have you here. This is the starting point for joining and contributing to the StarRocks community- improving docs, contributing code, giving talks, etc. For open-source first-timers, this repo also provides resources and guidance for you to complete your first pull request (PR). 


## What are our values?
Code is power. Community is strength. Together we create magic! 

[<img align="right" width="150" src="https://firstcontributions.github.io/assets/Readme/join-slack-team.png">](https://join.slack.com/t/starrocks/shared_invite/zt-z5zxqr0k-U5lrTVlgypRIV8RbnCIAzg)

## Communicating
The fastest way to learn is by participating! Join the conversations to get support and support others!

[Slack (EN)](https://join.slack.com/t/starrocks/shared_invite/zt-z5zxqr0k-U5lrTVlgypRIV8RbnCIAzg) |  [Forum (CN)](https://forum.starrocks.com/)  | [Issues](https://github.com/StarRocks/starrocks/issues) |🐦 [Twitter](https://twitter.com/StarRocksLabs)  | [Blogs & Events](https://www.starrocks.com/) 

</br>

## 🎯 Practice for your first pull request 
Whether you are an experienced developer or a code newbie, stepping into the world of open source and creating your first PR can be quite daunting. The guide here intends to help new contributors get familiar with GitHub workflow and learn how to contribute by simply adding your name to the [contributor.md](https://github.com/StarRocks/community/blob/main/Contributors.md).
Follow the steps below to learn how it works:

✨ Acknowledgment: This project is inspired by [firstcontributions](https://github.com/firstcontributions/first-contributions).

## Step 1: Fork this repository

<img align="right" width="300" src="https://github.com/kateshaowanjou/community-1/blob/main/Contributors/guide/fork.png" alt="fork this repository" />
Fork this repository by clicking on the fork button on the top of this page.
This will create a copy of this repository in your account.

## Step 2: Clone the repository

<img align="right" width="300" src="https://firstcontributions.github.io/assets/Readme/clone.png" alt="clone this repository" />

Now clone the forked repository to your machine. Go to your GitHub account, open the forked repository, click on the code button and then click the _copy to clipboard_ icon.

Open a terminal and run the following git command:

```
git clone "url you just copied"
```

where "url you just copied" (without the quotation marks) is the url to this repository (your fork of this project). See the previous steps to obtain the url.

<img align="right" width="300" src="https://github.com/kateshaowanjou/community-1/blob/main/Contributors/guide/clone.png" alt="copy URL to clipboard" />

For example:

```
git clone https://github.com/this-is-you/first-contributions.git
```

where `this-is-you` is your GitHub username. Here you're copying the contents of the first-contributions repository on GitHub to your computer.

## Step 3: Create a branch

Change to the repository directory on your computer (if you are not already there):

```
cd first-contributions
```

Now create a branch using the `git checkout` command:

```
git checkout -b your-new-branch-name
```

For example:

```
git checkout -b add-alonzo-church
```

(The name of the branch does not need to have the word _add_ in it, but it's a reasonable thing to include because the purpose of this branch is to add your name to a list.)

## Step 4: Make necessary changes and commit those changes

Now open `Contributors.md` file in a text editor, add your name to it. Don't add it at the beginning or end of the file. Put it anywhere in between. Now, save the file.

<img align="right" width="450" src="https://firstcontributions.github.io/assets/Readme/git-status.png" alt="git status" />

If you go to the project directory and execute the command `git status`, you'll see there are changes.

Add those changes to the branch you just created using the `git add` command:

```
git add Contributors.md
```

Now commit those changes using the `git commit` command:

```
git commit -m "Add <your-name> to Contributors list"
```

replacing `<your-name>` with your name.


## Step 5: Push changes to GitHub

Push your changes using the command `git push`:

```
git push origin <add-your-branch-name>
```

replacing `<add-your-branch-name>` with the name of the branch you created earlier.

## Step 6: Sign the CLA
We require contributors to sign [Contributor License Agreement (CLA)](https://cla-assistant.io/StarRocks/community?pullRequest=6)
when contributing to StarRocks. The checks will fail if you don't have it completed when submitting PR.

For more information, see what is [CLA](https://en.wikipedia.org/wiki/Contributor_License_Agreement).


## Step 7: Submit your changes for review

If you go to your repository on GitHub, you'll see a `Compare & pull request` button. Click on that button.

<img style="float: right;" src="https://github.com/kateshaowanjou/community-1/blob/main/Contributors/guide/create%20PR.png" alt="create a pull request" />

Now submit the pull request.

<img style="float: right;" src="https://github.com/kateshaowanjou/community-1/blob/main/Contributors/guide/open%20PR.png" alt="submit pull request" />

Soon your changes will be merging into the master branch. You will get a notification email once the changes have been merged.

## What's next?
Now you have successfully submitted your first PR, you can now head over to other repositories to continue contributing. However, the workflow and requirements may be a little different from our early practice. Read Contributing to StarRocks [Contributing to StarRocks](https://github.com/kateshaowanjou/starrocks/tree/main#contributing-to-starrocks) to learn more. 

Now you can:
1. Pick a [good first issue](https://github.com/StarRocks/starrocks/labels/good%20first%20issue) and start contributing
2. [File an issue](https://github.com/StarRocks/starrocks/issues), when problems and requirements are found
3. Improve docs
4. Share your experience with our community members by participating in talks, events, writing blogs, answering questions, etc.

