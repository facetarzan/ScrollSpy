# UniApp 实现 ScrollSpy 吸顶锚点导航

# 项目背景
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/6185fed98ec64f94a258d9c615303ca0.gif#pic_center =300x400)

最近看到其它产品有一个比较好用丝滑的功能，页面往下滑的时候顶部出现 Tab 栏，点哪个跳哪个，滑到哪个模块 Tab 自动高亮。
这个功能有个专业名字叫 ScrollSpy，翻译过来就是"滚动侦测导航"，很多 App 里都能见到，实现起来其实不复杂，今天把整个思路和代码记录一下。

# 思路梳理

整个功能拆成三块：
##  1.吸顶
UniApp 小程序端不能用 `position: sticky`，所以要用 `onPageScroll` 监听滚动，当 `scrollTop` 超过 Tab 栏原本的位置时，给 Tab 栏加 `position: fixed`，同时加一个占位 `view` 防止页面跳动。
## 2. 滚动高亮
 提前量好每个 section 距页面顶部的绝对距离存起来，滚动时拿 `scrollTop` 和这些距离比较，找到"最后一个已经滚过视口的 section"，就是当前应该高亮的 Tab。

核心逻辑就这几行：

```js
let active = tabs[0].id //默认首个tab高亮
tabs.forEach(tab => {
//循环遍历tabs，windowHeight为视口高度(uni.getSystemInfoSync().windowHeight)
  if (sectionTops[tab.id] <= scrollTop + windowHeight / 2) {
  // 这里选用节点距离顶部的距离是否小于当前滚动距离来切换tab
  // 觉得别扭的可以理解为当前滚动距离是否大于节点所在位置，如果大于说明节点出现了(至少出现过)
    active = tab.id  
  }
})
currentTab = active
```
为什么是遍历完才停，不是找到就 break？因为要找的是"所有已经滚进来的节点里面最靠下的"，比如三个都超过了视口顶部，如果中途 break 就只会拿到第一个满足条件的，不对。

## 3. 点击跳转

直接用存好的节点的绝对位置，减去视口的一半，==前面忘记强调了，这个值完全看你自己想让节点出现在视口的哪个位置，出现在顶部就减去tab栏，出现在中间就减去视口的一半==：

```js
 	currentTab.value = dataKey
 	//sectionTops是存储节点距离视口距离的数组
    const targetY = sectionTops.value[dataKey] - (windowHeight / 2)
    uni.pageScrollTo({
        scrollTop: targetY,
        duration: 300,
    })
```

---

# 有几个坑要注意

## 坑1：`boundingClientRect` 返回的是相对视口的距离

用 `uni.createSelectorQuery` 量节点位置时，`res.top` 是相对视口顶部的距离，不是页面绝对位置。如果页面已经滚动了一段，这个值会变小。

所以存的时候要加上当前 `scrollTop`，还原成页面绝对位置：

```js
sectionTops[tab.id] = res.top + currentScrollTop
```

不过最稳的做法是在页面刚加载、`scrollTop = 0` 的时候量取，这样 `res.top` 本身就等于页面绝对位置，不用加也没问题。

## 坑2：要在数据渲染完之后才能量位置

如果在请求回调外面量，section 还没渲染出来，`res` 会是 `null`。要在数据赋值之后，等 `nextTick` DOM 更新完再量：

```js
await handleGetDetail(id)
await nextTick()
measurePositions()
```

## 坑3：函数里没有 `return` Promise，外面 `await` 等不到
外面 await，如果没有returnpromise那么函数立刻返回 undefined，还是异步
```js
// ❌ 这样 await 等不到
const handleGetDetail = (id) => {
  request(id).then(res => {
    data.value = res.data
  })
}

// ✅ 加上 return
const handleGetDetail = (id) => {
  return request(id).then(res => {
    data.value = res.data
  })
}
```

**坑4：在 `<script setup>` 里用 `createSelectorQuery` 要加 `.in(instance.proxy)`**

不加的话小程序端经常返回 `null`：

```js
import { getCurrentInstance } from 'vue'
const instance = getCurrentInstance()

uni.createSelectorQuery()
  .in(instance.proxy)  // 加这个
  .select('#section-id')
  .boundingClientRect(res => { ... })
  .exec()
```

## 坑5：页面有图片时位置会变

图片加载完会撑高页面，之前量的位置就不准了。在图片的 `@load` 事件里重新量一次：

```html
<image :src="url" @load="measurePositions" />
```
## 坑6：不要节流，否则滚动距离会不准

当初也是觉得滚动一次触发几十次回调比较消耗性能，加了节流发现如果延时较长的话，滚动的距离有可能会截取，拿不到最大值

---
# 完整代码

```html
<template>
    <view class="page-container">
        <!-- 自定义导航栏 -->
        <view class="navbar">
            <view class="navbar-content">
                <view class="back-btn" @click="goBack">
                    <image class="back-icon" @load="measurePositions" src="/static/icons/back.png" mode="aspectFit"></image>
                </view>
                <text class="navbar-title">药膳食疗</text>
                <view class="placeholder"></view>
            </view>
        </view>
        <view style="height: 80rpx;"></view>
        <view style="background-color: #f9f7f1;position: relative;">
            <image class="header-img" @load="measurePositions" mode="widthFix" :src="serverUrl + dietDetail.image"></image>
        </view>
        <view class="content-container">
            <view class="tab-bar" :class="{ fixed: tabFixed }" id="tab-bar">
                <view v-for="tab in tabs" :key="tab.dataKey" class="tab-item" :class="{ active: currentTab === tab.dataKey }" @click="scrollToSection(tab.dataKey)">
                    {{ tab.label }}
                </view>
            </view>
            <!-- 各 section -->
           
            <view v-if="dietDetail.function" id="function" class="section">
                <view class="title">
                    <view style="background-color: rgb(139, 69, 19);width: 8rpx;height: 32rpx;">
                    </view>
                    <text>功效</text>
                </view>
                <view class="content">
                    <view v-for="item in dietDetail.function" :key="item">{{ item.content }}</view>
                </view>
            </view>
            <view v-if="dietDetail.foods" id="foods" class="section">
                <view class="title">
                    <view style="background-color: rgb(139, 69, 19);width: 8rpx;height: 32rpx;">
                    </view>
                    <text>食材</text>
                </view>
                <view class="content">
                    <view v-for="item in dietDetail.foods" :key="item">{{ item.content }}</view>
                </view>
            </view>
            <view v-if="dietDetail.manufacture" id="manufacture" class="section">
                <view class="title">
                    <view style="background-color: rgb(139, 69, 19);width: 8rpx;height: 32rpx;">
                    </view>
                    <text>制法</text>
                </view>
                <view class="content">
                    <view v-for="item in dietDetail.manufacture" :key="item">{{ item.content }}</view>
                </view>
            </view>

            <view class="bottom-space"></view>

        </view>
    </view>
</template>

```

```javascript
<script setup>
import { ref, onMounted, computed, getCurrentInstance, nextTick } from 'vue'
import { onLoad, onPullDownRefresh, onPageScroll } from "@dcloudio/uni-app"

const instance = getCurrentInstance()
const dietDetail = ref([])
const id = ref('')

// 模拟数据
const mockData = {
  id: 1,
  title: '测试数据title',
  source: '测试出处',
  classification: '汤',
  image: '',
  function: [
    { id: 1, content: '你好tarzan你好tarzan你好tarzan你好tarzan你好tarzan你好tarzan你好tarzan你好tarzan你好tarzan你好tarzan你好tarzan你好tarzan你好tarzan你好tarzan你好tarzan你好tarzan。' }
  ],
  foods: [
    { id: 2, content: '你好tarzan你好tarzan你好tarzan你好tarzan你好tarzan你好tarzan你好tarzan你好tarzan你好tarzan你好tarzan你好tarzan你好tarzan你好tarzan你好tarzan你好tarzan你好tarzan' }
  ],
  manufacture: [
    { id: 3, content: '你好tarzan你好tarzan你好tarzan你好tarzan你好tarzan你好tarzan你好tarzan你好tarzan你好tarzan你好tarzan你好tarzan你好tarzan你好tarzan你好tarzan你好tarzan你好tarzan' }
  ]
}

// 设备屏幕高度
const windowHeight = uni.getSystemInfoSync().windowHeight

// -------- 吸顶相关 --------
const tabFixed = ref(false)
const tabBarHeight = ref(44)
const navBarHeight = ref(0)
const tabOffsetTop = ref(150)

// -------- 高亮相关 --------
const currentTab = ref('')
const scrollIntoTabId = ref('')
const sectionTops = ref({})
const currentScrollTop = ref(0)

const tabConfig = [
  { id: 'function',    label: '功效', dataKey: 'function'    },
  { id: 'foods',       label: '食材', dataKey: 'foods'       },
  { id: 'manufacture', label: '制法', dataKey: 'manufacture' }
]

const tabs = computed(() => {
  return tabConfig.filter(tab => {
    const val = dietDetail.value[tab.dataKey]
    return val && val.length > 0 && val[0].content
  })
})


const handleGetDietDetail = (id) => {
  return new Promise((resolve) => {
    setTimeout(() => {
      dietDetail.value = mockData
      currentTab.value = tabs.value[0]?.id ?? ''
      resolve()
    }, 500)  // 模拟异步接口
  })
}

onMounted(() => {})

onLoad(async (options) => {
  id.value = options.id ?? '1'
  await handleGetDietDetail(id.value)
  await nextTick()
  measurePositions()
})

onPullDownRefresh(async () => {
  try {
    await handleGetDietDetail(id.value)
  } catch (err) {
  } finally {
    uni.stopPullDownRefresh()
  }
})

const goBack = () => {
  uni.navigateBack()
}

onPageScroll(({ scrollTop }) => {
  currentScrollTop.value = scrollTop
  tabFixed.value = scrollTop >= tabOffsetTop.value

  const offset = scrollTop + (windowHeight / 2)
  let active = tabs.value[0]?.id

  tabs.value.forEach(tab => {
    if ((sectionTops.value[tab.id] ?? Infinity) <= offset) {
      active = tab.id
    }
  })

  if (active && active !== currentTab.value) {
    currentTab.value = active
  }
})

const scrollToSection = (dataKey) => {
  currentTab.value = dataKey
  const targetY = sectionTops.value[dataKey] - (windowHeight / 2)
  uni.pageScrollTo({
    scrollTop: targetY,
    duration: 300,
  })
}

const measurePositions = () => {
  tabs.value.forEach(tab => {
    uni.createSelectorQuery()
      .in(instance.proxy)
      .select(`#${tab.id}`)
      .boundingClientRect(res => {
        if (!res) return
        sectionTops.value[tab.id] = res.top + currentScrollTop.value
      })
      .exec()
  })
}
</script>

```

```css
<style lang="scss" scoped>
.page-container {
    width: 100%;
    min-height: 100vh;
    height: 100%;
    background-color: #faf1df;

    .header-img {
        width: 540rpx;
        display: block;
        margin: 0 auto 0;
    }

    .navbar {
        position: fixed;
        top: 0;
        left: 0;
        right: 0;
        z-index: 100;
        padding-top: 80rpx;
        background-color: #fff;

        .navbar-content {
            display: flex;
            justify-content: space-between;
            align-items: center;
            height: 88rpx;
            padding: 0 30rpx;
        }

        .back-btn {
            width: 48rpx;
            height: 48rpx;
            display: flex;
            align-items: center;
            justify-content: center;
        }

        .back-icon {
            width: 32rpx;
            height: 32rpx;
        }

        .navbar-title {
            font-size: 34rpx;
            color: #333;
        }

        .placeholder {
            width: 48rpx;
        }
    }

    .content-container {
        padding: 0 30rpx;

        .tab-bar {
            display: flex;
            justify-content: flex-start;
            gap: 40rpx;
            color: rgba(179, 140, 53, 1);
            font-size: 32rpx;
            padding: 24rpx 30rpx;

            &.fixed {
                position: fixed;
                top: 150rpx;
                left: 0;
                right: 0;
                z-index: 100;
                background-color: #fff;
            }

            .tab-item.active {
                color: rgba(51, 51, 51, 1);
                padding: 0 0 15rpx 0;
                border-bottom: 5rpx solid rgba(179, 140, 53, 1);
            }
        }

        .name-section {
            background-color: #fff;
            margin: 16rpx 0;
            padding: 24rpx;
            box-sizing: border-box;
            box-shadow: 0rpx 8rpx 24rpx rgba(229, 211, 171, 0.2);
            border-radius: 10rpx;

            .title {
                font-size: 44rpx;
                color: #000;
                margin: 0 0 18rpx;
                border-bottom: 1rpx solid rgba(215, 215, 215, 1);
                padding: 0 0 20rpx;
            }

            .source {
                color: #000;
                display: flex;
                justify-content: flex-start;
                gap: 0 40rpx;
                font-size: 24rpx;

                .value {
                    color: rgba(163, 163, 163, 1);
                }
            }
        }

        .section {
            background-color: #fff;
            margin: 16rpx 0;
            padding: 24rpx;
            box-sizing: border-box;
            box-shadow: 0rpx 8rpx 24rpx rgba(229, 211, 171, 0.2);
            border-radius: 10rpx;
            color: rgba(163, 163, 163, 1);
            font-size: 24rpx;

            .title {
                display: flex;
                justify-content: flex-start;
                align-items: center;
                gap: 0 16rpx;
                color: rgba(51, 51, 51, 1);
                font-size: 28rpx;
                margin: 0 0 24rpx 0;
            }
        }

        .bottom-space {
            height: 2000rpx;
        }
    }
}
</style>

```
# 总结

整个 ScrollSpy 的核心就三件事：

1. **提前量好位置**：`createSelectorQuery` 拿到每个 section 的页面绝对位置存起来
2. **滚动时比较**：`scrollTop` 和存好的位置比，找最后一个滚过视口的
3. **点击时计算**：用存好的位置减去 Tab 高度，直接 `pageScrollTo`

说难不难，但坑不少，主要就是 `res.top` 是相对视口的这个点容易踩。搞清楚相对位置和绝对位置的区别，基本就没问题了。
