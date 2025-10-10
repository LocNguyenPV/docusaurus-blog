---
sidebar_position: 1
---

# Introduction to CI/CD

As a developer, at least one time you said or heard...

![img1](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/2zgmnbtmvl2ylww7691i.png)

Including me also 🤣 So why it happened? Hmmm...Many factors "contribute" to this, example:

- <u>Code conflict</u>
- <u>Integration error</u>
- <u>Process Manual</u>
- etc.

And CI/CD has born to helps us fix those issues

![img2](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/o55f17b8d6vrbqv58std.png)

## What is CI/CD

**CI/CD** are the short term for **Continuous Integration** & **Continuous Delivery/Deployment**. It helps the process become automated and deliver product to client in a reliable and fastest way.

![cicd-overview](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/02x94ic6u4o4hhvxqan4.png)

## Continuous Integration - CI

![ci-overview](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/0uvbhc34orh4yevvz4k9.png)

In **CI**, whenever a developer commit changes to the repository, it'll <u>build</u> & <u>run test automatically</u>. This helps ensure code is reliable and runnable, ready to integrate with the rest of the team's work. So with CI implementation, we can get the feedback right away after we commit, no need to wait the others for build and test

## Continuous Delivery/Deployment - CD

When your changes reach **CD**, the next phase after CI, it means they’re packaged, tested, and ready to be deployed to staging or production environments.

![congrats](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/tiwibfknffs9qmr96mw2.png)

In **CD**, there are two definitions.

- **Continuous Deliver:** Code is ready to deploy, but it need someone - maybe you, to click approve for deployment => best suitable for <u>production</u> environments

- **Continuous Deployment:** The code is automatically deployed to the target environment as soon as it passes all the checks, it's hands-free deployment => ideal for <u>staging</u> environments

![CD overview](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ktw2051g2nwpf8pywwfa.png)

## Conclusion

We've covered the basics of CI/CD. If your team/project hasn't implemented it yet, consider doing so, you'll likely find it beneficial. In the next post, I will cover some tips/tools can use in CI/CD.

Happy Coding!
