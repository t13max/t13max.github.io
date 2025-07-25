---
title: 测试
date: 2025-07-25 21:06:43
tags: [ Java,卡牌对战 ]
category: 测试
permalink: :year/:month/:day/:title/  
---


## 测试页面

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