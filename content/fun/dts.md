---
title: docker部署饥荒服务器
date: 2021-01-27T16:31:03+08:00
lastmod: 2021-02-08T12:31:03+08:00
author: sasaba
cover: /img/docker部署饥荒服务器.jpg
images:
  - /img/docker部署饥荒服务器.jpg
categories:
  - 游戏
tags:
  - 娱乐
  - docker
draft: true
---

使用docker部署饥荒服务器，实现无房主联机。

<!--more-->

## 步骤

### 申请饥荒服务器Token

打开游戏点击账户（个人资料），Generate Server Token获取令牌。

### 安装docker

**docker**

linux可以使用一键安装脚本

```shell
# centos
yum install -y curl
# debian/ubuntu
apt install -y curl

curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun
```

windows在官网下载Docker Desktop

https://hub.docker.com/editions/community/docker-ce-desktop-windows

推荐使用有一定带宽的linux服务器，因为家用带宽上传受限且udp性能不是很好。

**docker-compose**

linux

```shell
curl -L https://get.daocloud.io/docker/compose/releases/download/1.28.2/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
```

windows

安装了Docker Desktop后自带compose

### 获取饥荒的配置

可以自己先用联机版饥荒创建一个世界，把配置和mod选项都调好然后到 - 我的文档\Klei\DoNotStarveTogether\目录下找到创建的存档文件夹，比如Cluster1。

结构大概是这个样子的

```
MyDedicated-------------里面有2个文件夹和2个文件
    -- Caves ----------------- 有关地下洞穴的配置文件夹
          -- leveldataoverride.lua ------------地下洞穴世界资源配置
          -- modoverrides.lua ----------------地下洞穴mod配置
          -- server.ini -------------------------地下洞穴服务器配置
    -- Master -----------------有关地面的配置文件夹
         -- leveldataoverride.lua ------------地面世界资源配置
         -- modoverrides.lua ----------------地面mod配置
         -- server.ini -------------------------地面服务器配置
    -- cluster.ini ---------------服务器配置文件（必须用UTF-8无BOM格式编码）
    -- cluster_token.txt --------服务器令牌
```

### 配置docker-compose文件

使用的镜像为 [dstacademy/dontstarvetogether](https://hub.docker.com/r/dstacademy/dontstarvetogether)，它的说明还是比较完善的

先附上配置

docker-compose.yaml

```yaml
version: "2"
services:
  overworld:
    image: dstacademy/dontstarvetogether
    container_name: overworld
    hostname: overworld
    tty: true
    restart: always
    stdin_open: true
    command: dst-server start --update=all
    env_file: overworld.env
    environment:
      # Defined in YAML because multi-line support, but this can also be injected other ways
      # From a vanilla survival leveldataoverride.lua
      LEVELDATA_OVERRIDES: |
        return {
          desc="The standard Don't Starve experience.",
          hideminimap=false,
          id="SURVIVAL_TOGETHER",
          location="forest",
          max_playlist_position=999,
          min_playlist_position=0,
          name="Default",
          numrandom_set_pieces=4,
          ordered_story_setpieces={ "Sculptures_1", "Maxwell5" },
          override_level_string=false,
          overrides={
            alternatehunt="default",
            angrybees="default",
            antliontribute="default",
            autumn="default",
            bearger="default",
            beefalo="default",
            beefaloheat="default",
            bees="default",
            berrybush="default",
            birds="default",
            boons="default",
            branching="default",
            butterfly="default",
            buzzard="default",
            cactus="default",
            carrot="default",
            catcoon="default",
            chess="default",
            day="default",
            deciduousmonster="default",
            deerclops="default",
            disease_delay="default",
            dragonfly="default",
            flint="default",
            flowers="default",
            frograin="default",
            goosemoose="default",
            grass="default",
            houndmound="default",
            hounds="default",
            hunt="default",
            krampus="default",
            layout_mode="LinkNodesByKeys",
            liefs="default",
            lightning="default",
            lightninggoat="default",
            loop="default",
            lureplants="default",
            marshbush="default",
            merm="default",
            meteorshowers="default",
            meteorspawner="default",
            moles="default",
            mushroom="default",
            penguins="default",
            perd="default",
            petrification="default",
            pigs="default",
            ponds="default",
            prefabswaps_start="default",
            rabbits="default",
            reeds="default",
            regrowth="default",
            roads="default",
            rock="default",
            rock_ice="default",
            sapling="default",
            season_start="default",
            specialevent="default",
            spiders="default",
            spring="default",
            start_location="default",
            summer="default",
            tallbirds="default",
            task_set="default",
            tentacles="default",
            touchstone="default",
            trees="default",
            tumbleweed="default",
            walrus="default",
            weather="default",
            wildfires="default",
            winter="default",
            world_size="huge",
            wormhole_prefab="wormhole"
          },
          random_set_pieces={
            "Sculptures_2",
            "Sculptures_3",
            "Sculptures_4",
            "Sculptures_5",
            "Chessy_1",
            "Chessy_2",
            "Chessy_3",
            "Chessy_4",
            "Chessy_5",
            "Chessy_6",
            "Maxwell1",
            "Maxwell2",
            "Maxwell3",
            "Maxwell4",
            "Maxwell6",
            "Maxwell7",
            "Warzone_1",
            "Warzone_2",
            "Warzone_3"
          },
          required_prefabs={ "multiplayer_portal" },
          substitutes={  },
          version=3
        }
    ports:
      - "10999:10999/udp"
    volumes:
      - ./overworld:/var/lib/dsta/cluster
  underworld:
    image: dstacademy/dontstarvetogether
    container_name: underworld
    hostname: underworld
    tty: true
    stdin_open: true
    restart: always
    command: dst-server start --update=all
    env_file: underworld.env
    environment:
      # Defined in YAML because multi-line support, but this can also be injected other ways
      # From a vanilla caves leveldataoverride.lua
      LEVELDATA_OVERRIDES: |
        return {
          background_node_range={ 0, 1 },
          desc="Delve into the caves... together!",
          hideminimap=false,
          id="DST_CAVE",
          location="cave",
          max_playlist_position=999,
          min_playlist_position=0,
          name="The Caves",
          numrandom_set_pieces=0,
          override_level_string=false,
          overrides={
            banana="default",
            bats="default",
            berrybush="default",
            boons="default",
            branching="default",
            bunnymen="default",
            cave_ponds="default",
            cave_spiders="default",
            cavelight="default",
            chess="default",
            disease_delay="default",
            earthquakes="default",
            fern="default",
            fissure="default",
            flint="default",
            flower_cave="default",
            grass="default",
            layout_mode="RestrictNodesByKey",
            lichen="default",
            liefs="default",
            loop="default",
            marshbush="default",
            monkey="default",
            mushroom="default",
            mushtree="default",
            petrification="default",
            prefabswaps_start="default",
            reeds="default",
            regrowth="default",
            roads="never",
            rock="default",
            rocky="default",
            sapling="default",
            season_start="default",
            slurper="default",
            slurtles="default",
            start_location="caves",
            task_set="cave_default",
            tentacles="default",
            touchstone="default",
            trees="default",
            weather="default",
            world_size="default",
            wormattacks="default",
            wormhole_prefab="tentacle_pillar",
            wormlights="default",
            worms="default"
          },
          required_prefabs={ "multiplayer_portal" },
          substitutes={  },
          version=3
        }
    ports:
      - "11000:11000/udp"
    links:
      - overworld
    volumes:
      - ./underworld:/var/lib/dsta/cluster
```
overworld.env
```
TOKEN={服务器token}
MAX_PLAYERS=6
NAME=sasaba的世界
DESCRIPTION=sasaba的世界
PASSWORD=199564
MODS=1013308168,347079953,353697884,358749986,362175979,374550642,375850593,375859599,382177939,396026892,458940297,466732225,541537428,572538624
PAUSE_WHEN_EMPTY=true
SHARD_ENABLE=true
SHARD_NAME=overworld
SHARD_IS_MASTER=true
SHARD_MASTER_IP=overworld
SHARD_CLUSTER_KEY=AIgiahSOAG26471gakshgiSA1935G
```
underworld.env

```
TOKEN={服务器token}
MAX_PLAYERS=6
NAME=sasaba的地下世界
DESCRIPTION=sasaba的地下世界
PASSWORD=199564
MODS=1013308168,347079953,353697884,358749986,362175979,374550642,375850593,375859599,382177939,396026892,458940297,466732225,541537428,572538624
PAUSE_WHEN_EMPTY=true
SHARD_ENABLE=true
SHARD_NAME=underworld
SHARD_IS_MASTER=false
SHARD_MASTER_IP=overworld
SHARD_CLUSTER_KEY=AIgiahSOAG26471gakshgiSA1935G
```



modoverrides.lua

```lua
return {
  ["workshop-1013308168"]={
    ["configuration_options"]={ ["fast_eat"]="On", ["fast_pick_build_harvest_heal"]="On" },
    ["enabled"]=true 
  },
  ["workshop-347079953"]={
    ["configuration_options"]={ ["DFV_Language"]="EN", ["DFV_MinimalMode"]="default" },
    ["enabled"]=true 
  },
  ["workshop-353697884"]={ ["configuration_options"]={  }, ["enabled"]=true },
  ["workshop-358749986"]={
    ["configuration_options"]={ ["IndicatorSize"]=3, ["MaxIndicator"]=7000, ["PlayerIndicators"]=1 },
    ["enabled"]=true 
  },
  ["workshop-362175979"]={ ["configuration_options"]={ ["Draw over FoW"]="disabled" }, ["enabled"]=true },
  ["workshop-374550642"]={ ["configuration_options"]={ ["MAXSTACKSIZE"]=99 }, ["enabled"]=true },
  ["workshop-375850593"]={ ["configuration_options"]={  }, ["enabled"]=true },
  ["workshop-375859599"]={
    ["configuration_options"]={
      ["divider"]=5,
      ["random_health_value"]=0,
      ["random_range"]=0,
      ["show_type"]=0,
      ["unknwon_prefabs"]=1,
      ["use_blacklist"]=true 
    },
    ["enabled"]=true 
  },
  ["workshop-382177939"]={
    ["configuration_options"]={ [""]=true, ["eightxten"]="5x16", ["workit"]="yep" },
    ["enabled"]=true 
  },
  ["workshop-396026892"]={ ["configuration_options"]={ ["OPT_DIFFICULTY"]=1 }, ["enabled"]=true },
  ["workshop-458940297"]={
    ["configuration_options"]={
      ["DFV_ClientPrediction"]="default",
      ["DFV_FueledSettings"]="default",
      ["DFV_Language"]="EN",
      ["DFV_MinimalMode"]="default",
      ["DFV_PercentReplace"]="default",
      ["DFV_ShowACondition"]="default",
      ["DFV_ShowADefence"]="default",
      ["DFV_ShowAType"]="default",
      ["DFV_ShowDamage"]="default",
      ["DFV_ShowFireTime"]="default",
      ["DFV_ShowInsulation"]="default",
      ["DFV_ShowTemperature"]="default",
      ["DFV_ShowUses"]="default" 
    },
    ["enabled"]=true 
  },
  ["workshop-466732225"]={ ["configuration_options"]={  }, ["enabled"]=true },
  ["workshop-541537428"]={ ["configuration_options"]={  }, ["enabled"]=true },
  ["workshop-572538624"]={
    ["configuration_options"]={
      ["IS_CHS_ALL_MOD"]=true,
      ["IS_CHS_CHARACTER"]=true,
      ["IS_CHS_FIX_ALL"]=true,
      ["IS_CHS_SETTINGS"]=true 
    },
    ["enabled"]=true 
  } 
}
```
start.sh
```shell
#!/bin/bash

export MODS_OVERRIDES=$(< modoverrides.lua)
docker-compose up -d
```

介绍下每个文件的作用：

1. docker-compose用来启动两个容器分别来运行地上和地下的服务器，他主要需要配置的地方是env_file和environment，env_file指向了overworld.env和underworld.env主要是配置服务器的一些内容，比如服务器名字，描述，密码，是否暂停当没人的时候，同时配置其他世界的一些信息。environment主要是配置leveldataoverride.lua和modoverrides.lua，事实上他们可以通过start.sh里面导入环境变量的方式实现，我这里只是用了两种方法，LEVELDATA_OVERRIDES：和LEVELDATA_OVERRIDES=$(< leveldataoverride.lua)是等效的。
2. overworld.env和underworld.env主要配置服务器的信息（比如模组）和另外世界的信息：

```
# 为空时暂停
PAUSE_WHEN_EMPTY=true
# 启动其他的世界
SHARD_ENABLE=true
# 世界id
SHARD_NAME=overworld
# 是否为master
SHARD_IS_MASTER=true
# master id
SHARD_MASTER_IP=overworld
# 跟其他世界鉴权时用只要两个世界都是这个就行
SHARD_CLUSTER_KEY=AIgiahSOAG26471gakshgiSA1935G
```

3. modoverrides.lua和leveldataoverride.lua世界信息和模组配置，可以使用创建世界自动生成。
4. start.sh启动的脚本，可以把modoverrides.lua和leveldataoverride.lua导入环境变量，然后启动容器就可以了。

