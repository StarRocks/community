# StarRocks github workflow

The starrocks project uses the GitHub workflow overview, which includes some tips and suggestions, such as keeping the local environment 
synchronized and submitted with the upstream. This document provides a workflow for completing starrocks development on GitHub.

## Step1. Fork starrocks in the cloud
+ Visit: https://github.com/StarRocks/starrocks.  
+ Click `Fork` button (top right) to establish a cloud-based fork.

## Step2. Clone fork to local storage  
+ Click code > clone  
```
$ cd $working_dir  
$ git clone https://github.com/$user/starrocks
```
+ Add your cloned repo to upstream  
```
$ cd $working_dir/starrocks
$ git remote add upstream https://github.com/StarRocks/starrocks.git
```
+ Use the `git remote -v` command to view the remote warehouse
```
origin    https://github.com/$user/starrocks.git (fetch)
origin    https://github.com/$user/starrocks.git (push)
upstream  https://github.com/StarRocks/starrocks (fetch)
upstream  https://github.com/StarRocks/starrocks (push)
```
## Step3. Sync branch
+ Make sure your branch and remote content are consistent
```
$ cd $working_dir/starrocks
$ git checkout main
$ git fetch main
$ git rebase upstream/main
$ git push origin main 
```
## Step4 Create branch
+ Create branch based on master  
 `$ git checkout -b myfeature`
 
## Step5 Setting up the development environment
+ [Setting up the FE development environment](https://github.com/StarRocks/community/blob/main/Contributors/guide/IDEA.md)
+ [Setting up the BE development environmen](https://github.com/StarRocks/community/blob/main/Contributors/guide/Clion.md)
 
## Step6 Modify content or code
+ Now you can modify the content or code in your newly created branch

## Step7 Commit
+ Commit your changes
```
$ git add <filename>
$ git commit -m "$add a comment"
```
+ After commit your changes, you may need to modify and submit several rounds. You can refer to the following commands
```
$ git add <filename> (used to add one file)
git add -A (add all changes, including new/delete/modified files)
git add -a -m "$add a comment" (add and commit modified and deleted files)
git add -u (add modified and deleted files, not include new files)
git add . (add new and modified files, not including deleted files)
```
## Step8 Push
+ After completing the change, you need to push the changed content to the remote repo of your fork.   
`$ git push origin myfeature`

## Step9 pull request
+ Visit the repository of your fork https://github.com/$user/starrocks.
+ Click `Compare & pull request`

## Step10: Review and Merge
After the PR is submitted, it will be reviewed by at least 2 reviewers. 
The reviewer need to confirm that it is correct, and the maintainers of StarRocks will merge your pull request after accepting the final changes.

## Step11: Wait and check the CI passes
![](https://github.com/StarRocks/community/blob/main/Contributors/guide/picture/ci.png)
Once the PR is submitted, StarRcoks will automatically trigger a CI where the Required check must pass, and if a check does not pass, you can click on the Details link to see the details.
