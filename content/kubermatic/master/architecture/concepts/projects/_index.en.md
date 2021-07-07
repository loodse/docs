+++
title = "Projects"
date = 2021-02-24T12:01:35+02:00
weight = 30

+++

A project organizes all your Kubermatic resources. You can think of a project as the organizing entity for what you're
building. A project consists of a set of members, SSH keys, service accounts and clusters.

![Project](/img/kubermatic/master/architecture/concepts/projects/project-content.png)

Kubermatic project has the following:

 - A project name, which you provide.
 - A project ID, generated automatically.
 - A project members, who can access the project.
 - A number of cluster, which belongs to the project.

![Project](/img/kubermatic/master/architecture/concepts/projects/project.png)

Each project ID is unique across Kubermatic. Project name can be used many times. 

### Projects and permissions
For each project, you use three security groups (owners, editors, viewers) to grant the ability to manage and work on your project.
When you grant a role at the project level, the access provided by the role applies to object within the project.


