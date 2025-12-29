---
title: 首页
slug: ""
url: "/"  # 绑定根路径，核心正确
layout: "page"  # 新增：强制用单页面模板，避免渲染成文章列表
draft: false    # 新增：确保页面能被渲染（非草稿）
menu:
    main:
        weight: -120  # 权重越低，导航栏里越靠前，合理
        identifier: home  # 新增：和你 config.yaml 里的 home 标识符匹配
        name: 主页        # 新增：导航栏显示的文字，和 config 统一
        params:
            #icon: user
            icon: house-solid  # 图标正确（Stack 主题支持 Font Awesome 图标）

---

## my note