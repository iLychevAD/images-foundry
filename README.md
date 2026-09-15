# Images Foundry

## Problem

The software development lifecycle (SDLC) includes several actors, such as Kubernetes, GitLab CI, and microservices. Each of them may use container images, but today every actor maintains its own image list. The result is inconsistent and difficult to manage.

## Proposed solution

Use this repository as a single place to define all allowed container images.

- The `main` branch contains the image list.
- Anyone can propose a new image through a pull or merge request.
- The catalog format and its fields still need to be defined.

An initial YAML representation could look like this:

```yaml
images:
  - id: nodejs20
    description: Upstream Node.js 20 image with our updates.

  - id: kyverno
    description: Copy of the upstream Kyverno image.
```

## To decide

- Which fields should each image contain besides `id` and `description`?
- Should the catalog be one YAML file, or use a different structure?
