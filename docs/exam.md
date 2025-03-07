## Overview

Studying the fundamental concepts is important especially for the multiple-choice exams (KCNA and KCSA). The remaining three exams (CKAD, CKA, and CKS) are hands-on and require you to apply the concepts in a real-world scenario. It is important to practice often and be comfortable with navigating the Kubernetes environment using kubectl and being able to quickly reference the Kubernetes documentation as the exam is "open book".

## Navigating the Kubernetes Documentation

Understanding how the Kubernetes documentation site is structured and how to use its search functionality is important. 

!!! note
    During the exam, you can only access kubernetes.io documentation, GitHub documentation, and your exam portal. Stack Overflow, personal blogs, and other community forums are off-limits.

The Kubernetes documentation is located at [https://kubernetes.io/docs/](https://kubernetes.io/docs/). The search functionality is located at the top of the page. You can search for specific topics or use the search to find specific commands or resources. The site taxonomy is located on the left side of the page and is broken down into the following sections:

- **Concepts**: Covers fundamental concepts; ideal for beginners.
- **Tasks**: Provides instructions on specific tasks; useful for practical guidance.
- **Tutorials**: Offers step-by-step guides for tasks.
- **Reference**: Contains the Kubernetes API reference for direct interaction with the API.

Often when you need to lean on a document to help with a task, you'll find yourself in the "Tasks" section. The site taxonomy will also be reflected in the page URL within the search results. So be mindful of the link you click on when you are searching for help.

The "Concepts" sections will provide thorough explanations of the fundamental concepts of Kubernetes. This is good content to lean on if you need examples with explanation of how things work. Just be mindful of the time you spend reading through the documentation. You want to be able to quickly find the information you need and move on.

## Practice, Practice, Practice

With the hands-on exams, practice is key. You will spend all your time in a terminal. Therefore, being proficient with the bash shell is crucial for success. Make sure you are comfortable with the following tools:

- vim
- sed
- systemctl
- journalctl
- ps

Use resources like [this vim cheat sheet](https://vim.rtorr.com/) to build muscle memory for essential commands.

These are tools that will help you navigate the environment and troubleshoot issues. You should also be comfortable with:

- redirecting output to files
- using pipes to grep output and search for specific information especially when troubleshooting
- using `kubectl explain` to understand resources

With your [local Kubernetes cluster](https://pauldotyu.github.io/kubestronaut/kubernetes/), you should try to go through all the "Tutorials" found in the Kubernetes documentation. 

Also, when you sign-up for the exam, you'll gain access to [killer.sh](https://killer.sh/) which is a platform that will allow you to take practice exams in a timed environment. You will get two opportunities to take the practice exam and you should take advantage of this. But be strategic about it. Don't take the practice exam until you've studied the material. Practice in a local kubernetes environment as much as possible, then take a practice exam when you feel like you are ready. The practice exam will give you step-by-step solutions so you can use that as additional study material. My advice is to save the last practice exam for a few days before the actual exam. This will give you good practice and you can use the solutions as a refresher.

## Exam Day

The tests are proctored remotely so it is important to log into the exam platform 15-20 minutes early to make sure everything is up to snuff. You will need to have a webcam and microphone so the proctor can see you and hear you. The proctors will ask you to show them your entire room to verify you're alone and have no unauthorized materials. Ensure you have a clean workspace and a camera that can be easily moved around for this purpose.

During the exam, time will be a major factor. These exams are designed in a way that it is nearly impossible to complete all the questions in the time allotted. So you'll need to be able to quickly identify the questions you can answer and move on from the ones you can't. This is where practice comes in. The more you practice, the more you'll be able to quickly identify the questions you can answer and move on from the ones you can't. Don't be discouraged if you don't get to answer every single question. I've cleared exams where I didn't answer every question so it is possible to pass without answering every question. Lastly, every keystroke matters so be familiar with short resource names and be able to quickly type them out. For example, `kubectl get pods -n kube-system` can be shortened to `k get po -n kube-system` (after setting up the kubectl alias).

If you have the luxury of using a large monitor, I would recommend it. This will allow you to have multiple browser windows and multiple terminal windows open at the same time. This will allow you to quickly reference the Kubernetes documentation and the exam questions at the same time.

By following these guidelines and dedicating ample practice time, you'll be well-prepared for the Kubernetes certification exams.
