---
date: '2025-08-10T13:38:02-04:00'
draft: true
title: 'Why Tauri v2 and Red Hat don’t mix…yet'
---

### The push and pull between modern dev libraries and enterprise operating systems - with a developer caught in the middle

My company has recently started introducing Rust into our codebase through way of Tauri applications for some of our graphical interfaces. This allows us to write React code for the front end, and Rust for the backend.  We can connect to our larger codebase in C++ using the cxx crate (link here).  With the introduction of Tauri version 2.0, we wanted to migrate to the latest version to take advantage of its new features, which include options to compile mobile operating systems in addition to desktop environments.

I took the initiative of testing migration with one of the applications i am responsible for. While Tauri’s migration docs are well-written, there were many code changes required, some of which were unintuitive.  The new permissions model, while powerful, offers an overly granular level of control for some of the simple applications we’ve built.  I completed the migration, and the application worked well in my development testing on Windows. But then I hit a larger hurdle - the fact that we ship our apps on both Windows and Linux, and many enterprise clients use Red Hat Enterprise 8 (RHEL8) for their Linux machines. Attempting a Tauri build on RHEL8 revealed that a few OS libraries were below the minimum supported version for Tauri version 2. 

This was disappointing, but not overly so. Tauri version 1 is a powerful framework, and we can achieve most any goals in our roadmap. I preserved my app’s migration in a GitHub branch so that we could come back to this when Red Hat 9 becomes the minimum supported version for enterprise Linux deployments. However - this might not be a while - Red Hat 8 end of life is not until 2029!

We could theoretically build on RHEL9 to enable clients who are upgrading their operating systems sooner.  I revisited my working branch this week on a RHEL9 machine, only to find that the OS libraries (namely, WebKit, libsoup, and Glib) did not meet Tauri version 2 minimum requirements. 

### Differing Goals

This problem reveals different approaches to development represented by the Tauri and Red Hat teams. Tauri is pushing modern development using languages such as Typescript and Rust, utilizing the latest Linux libraries to aid in development of next-generation applications. Red Hat takes the opposite view, focusing on stability and minimizing the chance of vulnerabilities of breaking changes. This makses sense, as Red Hat’s focus is on deployments in enterprise clients - they don’t want to update a system library until it’s been verified and tested for years in other operating systems.

For our use cases, we’ll continue to use Tauri version 1 for application development. This version will continue to get bug fixes and security patches, and the framework is flexible enough that we can build any feature that we have in mind.  And when Red Hat 10 becomes the de facto enterprise Linux OS, we’ll be ready to migrate to Tauri 2!

