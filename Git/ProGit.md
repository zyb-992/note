# ProGit

## Getting Started

#### Git Has Integrity

1. The mechanism that Git uses for this checksumming is called a **SHA-1** hash, This is a 40-character string composed of hexadecimal characters (0–9 and a–f) and calculated based on the contents of a file or directory structure in Git

#### The Three States

1. Pay attention now — here is the main thing to remember about Git if you want the rest of your learning process to go smoothly. Git has three main states that your files can reside in: modified, staged, and committed

   **• Modified means that you have changed the file but have not committed it to your database yet.** 

   **• Staged means that you have marked a modified file in its current version to go into your next commit snapshot.** 

   **• Committed means that the data is safely stored in your local database**

![image-20221023222943515](https://raw.githubusercontent.com/zyb-992/Photobed/master/zyb/202210232229633.png)

2. **The basic Git Workflow** goes something like this: 
   1. You modify files in your working tree. 
   2. You selectively stage just those changes you want to be part of your next commit, which adds only those changes to the staging area. 
   3. You do a commit, which takes the files as they are in the staging area and stores that snapshot permanently to your Git directory.