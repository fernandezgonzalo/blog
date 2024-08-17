---
title: 'The Twelve-Factor App'
date: 2024-08-07T22:22:06-03:00
draft: false
tags: ["Best Practices", "Software Development", "Software Design", "Code Quality", "Code Maintenance", "Programming Principles"]

resources:
- name: "featured-image"
  src: "twelve-factor-image.png"
---

# The Twelve-Factor App

The Twelve-Factor App methodology is a set of best practices for building modern web applications. It was developed by Heroku to help developers create applications that are portable, resilient, and scalable. This methodology is particularly useful for applications deployed as services and follows twelve principles that guide the development, deployment, and maintenance of web applications.

## I. Codebase

**One codebase tracked in version control, many deploys**

A twelve-factor app is always tracked in a version control system such as Git, Mercurial, or Subversion. A single codebase is used for all environments (development, staging, production), ensuring that the code deployed is the same across all stages. Multiple deployments of the app, such as staging and production, share the same codebase but may have different configurations.

## II. Dependencies

**Explicitly declare and isolate dependencies**

A twelve-factor app never relies on system-wide packages. It explicitly declares all dependencies in a dependency declaration manifest, such as `requirements.txt` for Python or `package.json` for Node.js. These dependencies are installed during the build process, ensuring consistency across environments. Dependency isolation tools like virtual environments in Python or Node's `npm` are used to prevent conflicts between libraries.

## III. Config

**Store config in the environment**

Configuration that can vary between deployments should be stored in environment variables. This includes credentials, resource handles, and other settings. By separating config from code, the same codebase can be used across different environments without modification. Environment variables can be managed using tools like `dotenv` or platform-specific solutions like Heroku's config vars.

## IV. Backing Services

**Treat backing services as attached resources**

Backing services, such as databases, message queues, or caches, should be treated as external resources. They should be accessed via URLs or other service locators stored in the environment variables. This approach allows services to be easily swapped out without code changes, promoting portability and resilience.

## V. Build, Release, Run

**Strictly separate build and run stages**

The build stage converts the code repository into an executable bundle. The release stage combines the build with the config to create a release that can be deployed. The run stage runs the app in the execution environment. By separating these stages, it ensures that code changes and config changes are tested and deployed consistently.

## VI. Processes

**Execute the app as one or more stateless processes**

A twelve-factor app runs as one or more stateless processes. Any data that needs to persist should be stored in a stateful backing service, such as a database. This makes scaling out the app easier, as processes can be added or removed without affecting the app's state. It also ensures that the app can recover gracefully from crashes.

## VII. Port Binding

**Export services via port binding**

A twelve-factor app is self-contained and does not rely on the runtime injection of a web server into the execution environment. It exports HTTP or other protocols via port binding, making it possible to run the app as a service that can be accessed over the network. This enables the app to be deployed in various environments, from local development to cloud platforms.

## VIII. Concurrency

**Scale out via the process model**

Twelve-factor apps are designed to scale out by running multiple processes. Processes can be of different types, such as web processes, worker processes, and scheduled tasks. By using process managers or container orchestration tools like Kubernetes, it is easy to scale the app horizontally and distribute the load across multiple instances.

## IX. Disposability

**Maximize robustness with fast startup and graceful shutdown**

Processes in a twelve-factor app are disposable, meaning they can be started or stopped at a moment's notice. Fast startup and graceful shutdown are crucial to ensure that the app can scale rapidly and handle changes in load. Processes should be able to terminate gracefully, allowing ongoing requests to complete and releasing any resources they hold.

## X. Dev/Prod Parity

**Keep development, staging, and production as similar as possible**

A twelve-factor app minimizes the gap between development and production environments. This includes using the same backing services, configuration methods, and dependency versions across all stages. By keeping environments as similar as possible, it reduces the risk of bugs that only appear in production and simplifies the deployment process.

## XI. Logs

**Treat logs as event streams**

Logs in a twelve-factor app are treated as event streams, with each process writing its event stream unbuffered to `stdout`. These logs can then be collected, aggregated, and analyzed by external tools. This approach decouples the app from the log management system, making it easier to scale and maintain.

## XII. Admin Processes

**Run admin/management tasks as one-off processes**

Admin tasks, such as database migrations or data cleanup scripts, should be run as one-off processes in the same environment as the app. These tasks should be executed using the same codebase and config, ensuring consistency and reducing the risk of errors.

---

The Twelve-Factor App methodology provides a robust framework for building scalable, maintainable, and portable web applications. By following these principles, developers can ensure that their applications are resilient to change and capable of running in a variety of environments.

For more information, visit the [Twelve-Factor App](https://12factor.net) website.