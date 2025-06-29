<script setup>
import { useRouter, useRoute } from 'vue-router'
import { usePlayerStore } from './stores/player'
import { h, ref } from 'vue'
import { NIcon, darkTheme } from 'naive-ui'
import { BookOutlined, ExperimentOutlined, CompassOutlined, TrophyOutlined, SettingOutlined, MedicineBoxOutlined, GiftOutlined, HomeOutlined, SmileOutlined, AppstoreOutlined, BugOutlined } from '@ant-design/icons-vue'
import { Moon, Sunny, Flash } from '@vicons/ionicons5'
import { getRealmName } from './plugins/realm'

const router = useRouter()
const route = useRoute()
const playerStore = usePlayerStore()
const spiritWorker = ref(null)
const menuOptions = ref([])
const isNewPlayer = ref(false)
const isLoading = ref(true) // 添加加载状态

// 初始化数据加载
playerStore.initializePlayer().then(() => {
  isLoading.value = false
  getMenuOptions()
})

// 监听玩家状态
watch(() => playerStore.isNewPlayer, (bool) => {
  isNewPlayer.value = bool
  if (!bool && route.path === '/') {
    router.push('/cultivation')
  }
})

// 灵力获取相关配置
const baseGainRate = 10000  // 基础灵力获取率

const getMenuOptions = () => {
  menuOptions.value = [,
    ...(isNewPlayer.value ? [{
      label: '欢迎',
      key: '',
      icon: renderIcon(HomeOutlined)
    }] : []),
    {
      label: '修炼',
      key: 'cultivation',
      icon: renderIcon(BookOutlined)
    },
    {
      label: '背包',
      key: 'inventory',
      icon: renderIcon(ExperimentOutlined)
    },
    {
      label: '抽奖',
      key: 'gacha',
      icon: renderIcon(GiftOutlined)
    },
    {
      label: '炼丹',
      key: 'alchemy',
      icon: renderIcon(MedicineBoxOutlined)
    },
    {
      label: '探索',
      key: 'exploration',
      icon: renderIcon(CompassOutlined)
    },
    {
      label: '秘境',
      key: 'dungeon',
      icon: renderIcon(Flash)
    },
    {
      label: '成就',
      key: 'achievements',
      icon: renderIcon(TrophyOutlined)
    },
    {
      label: '设置',
      key: 'settings',
      icon: renderIcon(SettingOutlined)
    },
    ...(playerStore.isGMMode ? [{
      label: 'GM调试',
      key: 'gm',
      icon: renderIcon(SmileOutlined)
    }] : []),
  ]
}
// 自动获取灵力
const startAutoGain = () => {
  if (spiritWorker.value) return
  spiritWorker.value = new Worker(new URL('./workers/spirit.js', import.meta.url))
  spiritWorker.value.onmessage = (e) => {
    if (e.data.type === 'gain') {
      playerStore.totalCultivationTime += 1
      playerStore.gainSpirit(baseGainRate)
    }
  }
  spiritWorker.value.postMessage({ type: 'start' })
}

onMounted(() => {
  startAutoGain()  // 启动自动获取灵力
})

// 图标
const renderIcon = (icon) => {
  return () => h(NIcon, null, { default: () => h(icon) })
}

// 获取当前路由对应的菜单key
const getCurrentMenuKey = () => {
  const path = route.path.slice(1) // 移除开头的斜杠
  return path // 如果是根路径，默认返回cultivation
}

// 菜单点击事件
const handleMenuClick = (key) => {
  router.push(`/${key}`)
}

</script>

<template>
  <n-config-provider :theme="playerStore.isDarkMode ? darkTheme : null">
    <n-message-provider>
      <n-dialog-provider>
        <n-spin :show="isLoading" description="正在加载游戏数据...">
          <n-layout>
            <n-layout-header bordered>
              <div class="header-content">
                <n-page-header>
                  <template #title>
                    我的放置仙途
                  </template>
                  <template #extra>
                    <n-button quaternary circle @click="playerStore.toggle">
                      <template #icon>
                        <n-icon>
                          <Sunny v-if="playerStore.isDarkMode" />
                          <Moon v-else />
                        </n-icon>
                      </template>
                    </n-button>
                  </template>
                </n-page-header>
                <n-scrollbar x-scrollable trigger="none">
                  <n-menu mode="horizontal" :options="menuOptions" :value="getCurrentMenuKey()"
                    @update:value="handleMenuClick" />
                </n-scrollbar>
              </div>
            </n-layout-header>
            <n-layout-content>
              <div class="content-wrapper">
                <n-card>
                  <n-space vertical>
                    <n-descriptions bordered>
                      <n-descriptions-item label="道号">
                        {{ playerStore.name }}
                      </n-descriptions-item>
                      <n-descriptions-item label="境界">
                        {{ getRealmName(playerStore.level).name }}
                      </n-descriptions-item>
                      <n-descriptions-item label="修为">
                        {{ playerStore.cultivation }} / {{ playerStore.maxCultivation }}
                      </n-descriptions-item>
                      <n-descriptions-item label="灵力">
                        {{ playerStore.spirit.toFixed(2) }}
                      </n-descriptions-item>
                      <n-descriptions-item label="灵石">
                        {{ playerStore.spiritStones }}
                      </n-descriptions-item>
                      <n-descriptions-item label="强化石">
                        {{ playerStore.reinforceStones }}
                      </n-descriptions-item>
                    </n-descriptions>
                    <n-collapse>
                      <n-collapse-item title="详细信息" name="1">
                        <n-divider>基础属性</n-divider>
                        <n-descriptions bordered :column="2">
                          <n-descriptions-item label="生命值">
                            {{ (playerStore.baseAttributes.health|| 0).toFixed(0) }}
                          </n-descriptions-item>
                          <n-descriptions-item label="攻击力">
                            {{ (playerStore.baseAttributes.attack|| 0).toFixed(0) }}
                          </n-descriptions-item>
                          <n-descriptions-item label="防御力">
                            {{ (playerStore.baseAttributes.defense|| 0).toFixed(0) }}
                          </n-descriptions-item>
                          <n-descriptions-item label="速度">
                            {{ (playerStore.baseAttributes.speed || 0).toFixed(0) }}
                          </n-descriptions-item>
                        </n-descriptions>
                        <n-divider>战斗属性</n-divider>
                        <n-descriptions bordered :column="3">
                          <n-descriptions-item label="暴击率">
                            {{ (playerStore.combatAttributes.critRate * 100).toFixed(1) }}%
                          </n-descriptions-item>
                          <n-descriptions-item label="连击率">
                            {{ (playerStore.combatAttributes.comboRate * 100).toFixed(1) }}%
                          </n-descriptions-item>
                          <n-descriptions-item label="反击率">
                            {{ (playerStore.combatAttributes.counterRate * 100).toFixed(1) }}%
                          </n-descriptions-item>
                          <n-descriptions-item label="眩晕率">
                            {{ (playerStore.combatAttributes.stunRate * 100).toFixed(1) }}%
                          </n-descriptions-item>
                          <n-descriptions-item label="闪避率">
                            {{ (playerStore.combatAttributes.dodgeRate * 100).toFixed(1) }}%
                          </n-descriptions-item>
                          <n-descriptions-item label="吸血率">
                            {{ (playerStore.combatAttributes.vampireRate * 100).toFixed(1) }}%
                          </n-descriptions-item>
                        </n-descriptions>
                        <n-divider>战斗抗性</n-divider>
                        <n-descriptions bordered :column="3">
                          <n-descriptions-item label="抗暴击">
                            {{ (playerStore.combatResistance.critResist * 100 || 0).toFixed(1) }}%
                          </n-descriptions-item>
                          <n-descriptions-item label="抗连击">
                            {{ (playerStore.combatResistance.comboResist * 100 || 0).toFixed(1) }}%
                          </n-descriptions-item>
                          <n-descriptions-item label="抗反击">
                            {{ (playerStore.combatResistance.counterResist * 100 || 0).toFixed(1) }}%
                          </n-descriptions-item>
                          <n-descriptions-item label="抗眩晕">
                            {{ (playerStore.combatResistance.stunResist * 100 || 0).toFixed(1) }}%
                          </n-descriptions-item>
                          <n-descriptions-item label="抗闪避">
                            {{ (playerStore.combatResistance.dodgeResist * 100 || 0).toFixed(1) }}%
                          </n-descriptions-item>
                          <n-descriptions-item label="抗吸血">
                            {{ (playerStore.combatResistance.vampireResist * 100 || 0).toFixed(1) }}%
                          </n-descriptions-item>
                        </n-descriptions>
                        <n-divider>特殊属性</n-divider>
                        <n-descriptions bordered :column="4">
                          <n-descriptions-item label="强化治疗">
                            {{ (playerStore.specialAttributes.healBoost * 100 || 0).toFixed(1) }}%
                          </n-descriptions-item>
                          <n-descriptions-item label="强化爆伤">
                            {{ (playerStore.specialAttributes.critDamageBoost * 100 || 0).toFixed(1) }}%
                          </n-descriptions-item>
                          <n-descriptions-item label="弱化爆伤">
                            {{ (playerStore.specialAttributes.critDamageReduce * 100 || 0).toFixed(1) }}%
                          </n-descriptions-item>
                          <n-descriptions-item label="最终增伤">
                            {{ (playerStore.specialAttributes.finalDamageBoost * 100 || 0).toFixed(1) }}%
                          </n-descriptions-item>
                          <n-descriptions-item label="最终减伤">
                            {{ (playerStore.specialAttributes.finalDamageReduce * 100 || 0).toFixed(1) }}%
                          </n-descriptions-item>
                          <n-descriptions-item label="战斗属性提升">
                            {{ (playerStore.specialAttributes.combatBoost * 100 || 0).toFixed(1) }}%
                          </n-descriptions-item>
                          <n-descriptions-item label="战斗抗性提升">
                            {{ (playerStore.specialAttributes.resistanceBoost * 100 || 0).toFixed(1) }}%
                          </n-descriptions-item>
                        </n-descriptions>
                      </n-collapse-item>
                    </n-collapse>
                    <n-progress type="line"
                      :percentage="Number(((playerStore.cultivation / playerStore.maxCultivation) * 100).toFixed(2))"
                      indicator-text-color="rgba(255, 255, 255, 0.82)" rail-color="rgba(32, 128, 240, 0.2)"
                      color="#2080f0" :show-indicator="true" indicator-placement="inside" processing />
                  </n-space>
                </n-card>
                <router-view />
              </div>
            </n-layout-content>
          </n-layout>
        </n-spin>
      </n-dialog-provider>
    </n-message-provider>
  </n-config-provider>
</template>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}
:root {
  --n-color: rgb(16, 16, 20);
  --n-text-color: rgba(255, 255, 255, 0.82);
}

html.dark {
  background-color: var(--n-color);
}

body {
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Oxygen,
    Ubuntu, Cantarell, "Open Sans", "Helvetica Neue", sans-serif;
}

.n-config-provider,
.n-layout {
  height: 100%;
  min-height: 100vh;
}

.header-content {
  max-width: 1200px;
  margin: 0 auto;
  padding: 0 16px;
}

.content-wrapper {
  max-width: 1200px;
  margin: 0 auto;
  padding: 16px;
}

.n-card {
  margin-bottom: 16px;
}

.footer-content {
  display: flex;
  justify-content: center;
  align-items: center;
  padding: 12px;
}

.n-page-header__title {
  padding: 16px 0;
  margin: 0 16px;
}

::-webkit-scrollbar {
  width: 12px;
  height: 12px;
}

::-webkit-scrollbar-track {
  background-color: rgba(0, 0, 0, 0.03);
}

::-webkit-scrollbar-thumb {
  background-color: rgba(0, 0, 0, 0.2);
  border-radius: 6px;
  border: 3px solid transparent;
  background-clip: padding-box;
}

::-webkit-scrollbar-thumb:hover {
  background-color: rgba(0, 0, 0, 0.3);
}

html.dark ::-webkit-scrollbar-track {
  background-color: rgba(255, 255, 255, 0.03);
}

html.dark ::-webkit-scrollbar-thumb {
  background-color: rgba(255, 255, 255, 0.2);
}

html.dark ::-webkit-scrollbar-thumb:hover {
  background-color: rgba(255, 255, 255, 0.3);
}
</style>
