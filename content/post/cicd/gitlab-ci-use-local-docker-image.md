---
title: "[gitlab ci] 怎么使用本地的docker image"
description: gitlab ci使用记录
date: 2024-09-02
categories:
    - CI/CD
tags:
    - CI/CD
    - gitlab
    - docker
---

每次跑gitlab ci都需要pull一个镜像很费时间，则可以修改gitlab-runner的参数

修改config.toml文件中的[[runners]] [runners.docker] pull_policy = "if-not-present"

```toml
[[runners]]
  [runners.docker]
    pull_policy = "if-not-present"
```

参考：

* <https://docs.gitlab.com/runner/executors/docker.html#how-pull-policies-work>
* <https://stackoverflow.com/questions/63043714/gitlab-runner-pulls-image-for-every-job>
