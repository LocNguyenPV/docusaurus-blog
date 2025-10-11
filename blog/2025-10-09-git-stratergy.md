---
sidebar_position: 2
---

<!-- truncate -->

# Git - Dev/DevOps best friend

In a software, **Git** is more than just a version control system - It's a <u>backbone</u> of the entire project. It acts as a <u>single source of truth</u>, it store anything related to code and infrastructure.

## Why Git is important

Git solves 3 big problems:

1. **History changes:** Every changes is recorded, making it easy to review, revert, or understand the evolution of the code.
2. **Automation workflow:** Git integrates seamlessly with CI/CD pipelines, enabling automated testing, building, and deployment.
3. **Made/keep an update:** Git ensures that everyone works with the latest version, reducing conflicts and improving team efficiency.

## Tips for using Git effectively

- **Small commit and clearly message:** You should commit a small part, not a big change and writing clearly message which describe what you did
- **Give a meaning name for branch:** Don't make your team member be confuse or guessing what the branch do
- **Keep the `main` branch stable:** Ensure code in `main` branch is always stable, even new members can clone and run it without error
- **Tag version:** Tagging version for rollback / manage easily

## Branching strategy in Git

Now a day, we have two popular strategies

- **Git flow**
- **Trunk-Based**

Let's take a look into it 👀

### Git flow

![git-flow](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/kaeaqv8k8vjw23citvih.png)

<figCaption>Git flow</figCaption>

In real-world projects, we often create many branches—and each branch serves a specific purpose. Depending on your company or team, branch names might differ, but they should still align with the core structure of Git Flow.

- `main`: production-ready branch, it contain stable and deployable code
- `hotfix`: Urgent fixes for production issues. Branched from `main` and merged into both `main` and `develop`
- `release`: Code ready for final testing before merging into `main`
- `develop`: The integration branch for development
- `features`: Developing new features, typical branched from `develop` and merged back when completed

> We can use git flow for a big team or projects have complex release cycles

### Trunk-Based

![trunk-based](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/jnizutokkybpa0sffy7s.png)

In trunk-based, we only have two branches `main` and `features`, with `features`'s life-time is smaller and shorter than <u>git flow</u> (usually 2-3 days). This strategy helps us take advantage of <u>automation </u> in git.

> We can use trunk-based for a small team or pet projects

## Conclusion

![meme](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/867f9dgrmfn55rd4el6f.png)

Choosing a right strategy will help your team deliver product faster and reliable. No need to follow the others, just choose your best suite 😎

Happy Coding!
