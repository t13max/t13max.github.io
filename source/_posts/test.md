---
title: 测试
date: 2025-07-25 21:06:43
tags: [ Java,卡牌对战 ]
category: 测试
keywords: 测试
permalink: :year/:month/:day/:title/
description: 一个测试页面
top_image:
cover: https://jsd.012700.xyz/gh/jerryc127/CDN@latest/cover/default_bg.png
related_post:
  enable: true
  limit: 6 # 显示推荐文章数目
  date_type: created # or created or updated 文章日期显示创建日或者更新日

---

## 测试页面

这是一个测试页面 下面是一段lua代码

```lua
function Timer.runAfter(interval, f, ...)
    local timerid = NewSession()
    local args = { farg = { ... }, timerid = timerid }

    local function run()
        if Timer.CheckSession(args.timerid) then
            f(table.unpack(args.farg))
        end
        RecoverSession(args.timerid)
    end

    skynet.timeout(interval, run)
    return timerid
end

```