You are correct that it's generally a good practice to ensure your branch is up-to-date with the main branch before creating a pull request. This helps avoid conflicts and ensures your changes are compatible with the latest codebase. Here’s a step-by-step process to update your branch and create a pull request:

1. **Switch to your main branch**:
   ```sh
   git checkout main
   ```

2. **Fetch the latest changes from the remote repository**:
   ```sh
   git fetch origin
   ```

3. **Pull the latest changes into your local main branch**:
   ```sh
   git pull origin main
   ```

4. **Switch back to your feature branch**:
   ```sh
   git checkout feature/117-new-feature
   ```

5. **Merge the latest changes from main into your feature branch**:
   ```sh
   git merge main
   ```

6. **Resolve any merge conflicts that arise**. If there are conflicts, Git will notify you, and you’ll need to manually resolve them in the affected files. After resolving conflicts, add the resolved files:
   ```sh
   git add <resolved_file>
   ```

7. **Complete the merge by committing the changes**:
   ```sh
   git commit
   ```

8. **Push the updated feature branch to the remote repository**:
   ```sh
   git push origin feature/117-new-feature
   ```

9. **Create a pull request** from your `feature/117-new-feature` branch to the `main` branch on your repository platform (e.g., GitHub, GitLab, Bitbucket).

This process ensures that your branch incorporates the latest updates from the main branch, reducing the likelihood of conflicts and making it easier for your pull request to be reviewed and merged.