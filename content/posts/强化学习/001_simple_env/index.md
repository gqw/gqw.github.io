---
date: 2021-11-16
title: "创建最简单的gym自定义环境"
author: asura9527 ([@gqw](https://gqw.github.io))
menu:
  sidebar:
    name: 创建最简单的gym自定义环境
    identifier: 001_create_simple_gym_env
    parent: reinforcement_learning
    weight: 10
hero: imgs/gym_env.png
---

## 问题

学习一样东西最好自己动手做一个，想明白OpenAI gym的运行机制，我们就从0开始搭建个环境。


## 解决步骤	

目录结构：

```bash
(.venv) gqw@u:~/workspace/gymtrain$ tree
.
├── env
│   ├── foo_env.py
│   └── __init__.py
└── foo_train.py

1 directory, 3 files
```

gym将代码分为两部分：环境部分和训练部分

### gym 环境

gym 对环境镜像了抽象，无论多复杂的环境，最终暴露出来的只有gym.Env抽象的5个方法：

```
        step
        reset
        render
        close
        seed
```

我们现在实现一个最简单的环境类FooEnv:

```
import gym

class FooEnv(gym.Env):
    """ This is a simple envirionment """

    def __init__(self) -> None:
        super(FooEnv).__init__()
        self.seed()

    def seed(self):
        print("seed")
        pass

    def step(self, action):
        print("step")
        reward = 0
        done = False
        return None, reward, done, {}

    def reset(self):
        print("reset")
        pass
    
    def render(self):
        return True

    def close(self):
        pass
```

它继承gym.Env并且实现其5个抽象接口方法。到这里gym的环境基础就写完了，就这么简单。



### 环境注册

注册也非常简单，只要在任何地方调用gym提供的register方法即可，通常我们会将注册写在环境目录下的`__init__.py`文件中，这样在导入环境类的时候便能自动注册了，就像下面这样：

```
from gym.envs.registration import register

register(
    id="Foo-v0",
    entry_point="env.foo_env:FooEnv"
)
```

这里的参数`id`便是后面训练时gym.make使用的名字，当然entry_point便是实际的类了。注意这里的`id`需要匹配一定的规则：^(?:[\w:-]+\/)?([\w:.-]+)-v(\d+)$.)， 也就是`{{名字}}-v{{版本号}}`格式。

### 训练

```
    env = gym.make('Foo-v0')
    env.reset()
    
    for _ in range(1000):
        # is_open = env.render()
        # if is_open == False:
        #     break

        _, _, done, _ = env.step({})
        if done == True:
            env.reset()
    env.close()
```

训练其实就是env暴露的那5个方法，其中最重要的就是step和reset方法了，其他像render和close是用于展示用的可以没有， seed一般在环境类的构造函数中直接调用了，所以剩下的只有step和reset了。



### 仓库代码

https://github.com/gqw/foo_train.git

## 参考信息

[1] ADITHYASOLAI. Cracking Blackjack — Part 2[EB/OL]. Medium, 2020-07-28.  https://towardsdatascience.com/cracking-blackjack-part-2-75e32363e38.

[2] FENG Y. Create a customized gym environment for Star Craft 2[EB/OL]. Medium, 2019-11-25.  https://towardsdatascience.com/create-a-customized-gym-environment-for-star-craft-2-8558d301131f.
