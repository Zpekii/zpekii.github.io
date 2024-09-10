---
title: create github.io page
author: Zpekii
tags: [github,page,jekyll]
categories: [github,page,jekyll]
description: 创建github.io page
---

# 创建github.io Page

**主题：** https://github.com/cotes2020/jekyll-theme-chirpy

**主题演示：** https://chirpy.cotes.page/

## 创建github仓库

- 创建名为 `username.github.io`的仓库,其中`username`为github账号名

## Fork主题仓库

- 访问github仓库`https://github.com/cotes2020/jekyll-theme-chirpy`, 点击上侧 "Fork" 按钮，选择已创建好的 `username.github.io`仓库，即可将主题仓库复制到自己的仓库

<img src="/assets/img/2024-9-10-CreateGithubPage.assets/image-20240911000858691.png" alt="image-20240911000858691" />

## 在容器环境内拉取`username.github.io`仓库并作初始化

- 通过Dockerfile创建专门的容器镜像

  - ```dockerfile
    # 使用官方ubuntu作为父镜像
    FROM ubuntu:latest
    
    # 设置镜像维护者信息
    LABEL maintainer="Zpekii" description="预安装各种工具、Ruby，用于拉取github.io仓库，用于发布github.io博客"
    
    # 预安装工具、Ruby
    RUN apt update && apt install -y \
    	vim \
    	nano \
    	curl	\
    	git	\
    	rsync \
    	ruby-full build-essential zlib1g-dev \
    	&& rm -rf /var/lib/apt/lists/*
    
    # 配置相关环境变量
    RUN echo '# Install Ruby Gems to ~/gems' >> ~/.bashrc
    RUN echo 'export GEM_HOME="$HOME/gems"' >> ~/.bashrc
    RUN echo 'export PATH="$HOME/gems/bin:$PATH"' >> ~/.bashrc
    
    # 安装 jekyll bundler
    RUN gem install jekyll bundler
    
    # 默认执行命令
    CMD ["/bin/bash"]
    ```

  - 在任意目录下创建文件名为`Dockerfile`的文件，拷贝上述内容至创建好的`Dockerfile`，接着在当前目录下运行终端，执行

    ```bash
    docker build -t ubuntu:jekyll .
    ```

    **注意:** 如果没有安装 `ubuntu:latest`镜像请先执行 `docker pull ubuntu:latest` 拉取

- 在`WSL`中执行以下内容（"XXX"内容需要自行替换实际路径）创建专门容器

  - ```bash
    sudo docker run -it --rm --name jekyll-shell \
            -v /mnt/c/XXX/username.github.io:/mnt/Project \
            -v data:/var/data \
            -v /var/run/docker.sock:/var/run/docker.sock \
            -v $HOME/.ssh:/root/.ssh \
            -v $HOME/bin/conf/node/profile:/root/.bashrc \
            -w /mnt/Project \
            -e DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=1 \
            --network qnear \
            ubuntu:jekyll bash
    ```

- 进入到`username.github.io`项目文件，执行

  - ```bash
    tools/init.sh
    ```

- 等待初始化完成

- 接着在根目录执行安装依赖

  - ```bash
    bundle
    ```

## 编辑配置文件(_config.yml)修改信息

- 打开项目根目录下的`_config.yml`
- 修改必要参数: 
  - `url` : `username.github.io`
  - `timezone`  `Asia/Shanghai`
  - `lang`: `zh-CN`
  - `github`:
    - `username`: 实际github账号名
  - `social`:
    - `links`: 实际的社交账号链接

## 开启评论

**使用github APP:** **gicus** [giscus](https://giscus.app/zh-CN)

- 需要满足以下条件:

  - 选择 giscus 连接到的仓库。请确保：
    1. **该仓库是[公开的](https://docs.github.com/en/github/administering-a-repository/managing-repository-settings/setting-repository-visibility#making-a-repository-public)**，否则访客将无法查看 discussion。
    2. **[giscus](https://github.com/apps/giscus) app 已安装**，否则访客将无法评论和回应。
    3. **Discussions** 功能已[在你的仓库中启用](https://docs.github.com/en/github/administering-a-repository/managing-repository-settings/enabling-or-disabling-github-discussions-for-a-repository)。

- 在 `仓库` 下输入框输入仓库 `账号名/仓库名` 可以进行验证是否满足条件

- 若满足，则进行下一步

- 在`Discussion分类`的下拉框中选择推荐 `Announcements`项，其他保持默认

- 最后在`启用giscus`中可以查看各个属性值，将属性值拷贝添加到`_config.yml`中的 `comments`下对应参数，以下是一个示例数据

  - giscus提供的数据

    - ```html
      <script src="https://giscus.app/client.js"
              data-repo="username/username.github.io"
              data-repo-id="R_kgXXXXXA"
              data-category="Announcements"
              data-category-id="DIC_kwXXXXXWf9"
              data-mapping="pathname"
              data-strict="0"
              data-reactions-enabled="1"
              data-emit-metadata="0"
              data-input-position="bottom"
              data-theme="preferred_color_scheme"
              data-lang="zh-CN"
              crossorigin="anonymous"
              async>
      </script>
      ```

      

  - _config.yml

    - ```yaml
      comments:
        # Global switch for the post-comment system. Keeping it empty means disabled.
        provider: giscus # [disqus | utterances | giscus]
        # The provider options are as follows:
        disqus:
          shortname: # fill with the Disqus shortname. › https://help.disqus.com/en/articles/1717111-what-s-a-shortname
        # utterances settings › https://utteranc.es/
        utterances:
          repo: # <gh-username>/<repo>
          issue_term: # < url | pathname | title | ...>
        # Giscus options › https://giscus.app
        giscus:
          repo: username/username.github.io # <gh-username>/<repo>
          repo_id: R_kgXXXXXA
          category: Announcements
          category_id: DIC_kwXXXXXWf9
          mapping: # optional, default to 'pathname'
          strict: # optional, default to '0'
          input_position: # optional, default to 'bottom'
          lang: zh-CN # optional, default to the value of `site.lang`
          reactions_enabled: # optional, default to the value of `1`
      ```

  ## 上传图片

  - 将图片路径改为(将图片移动到项目`assets/img`目录下) `/assets/img/YYYY-MM-DD-Title.assets/image.png`

  - 图片markdown语法改为`html`

  



​	
