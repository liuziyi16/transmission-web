<template>
  <div class="files-tab" style="min-width: 1024px">
    <div class="title">
      <n-text class="text-xs">
        {{ t('torrentDetail.files.fileCount') }}: {{ totalFiles }} | {{ t('torrentDetail.files.selectedDownload') }}: {{ selectedFiles }} | {{ t('torrentDetail.files.totalSize') }}: {{ formatSize(totalSize) }}
      </n-text>
      <div class="flex gap-2">
        <n-button size="small" @click="selectAll">{{ t('torrentDetail.files.selectAll') }}</n-button>
        <n-dropdown :options="priorityOptions" @select="handleBatchPriority" placement="bottom-end">
          <n-button size="small">{{ t('torrentDetail.files.batchSetPriority') }}</n-button>
        </n-dropdown>
      </div>
    </div>
    <div class="content">
      <n-tree
        ref="treeRef"
        :data="treeData"
        :checked-keys="checkedKeys"
        :render-label="renderLabel"
        :render-prefix="renderPrefix"
        :render-suffix="renderSuffix"
        checkable
        cascade
        check-strategy="child"
        @update:checked-keys="onCheckedKeysChange"
        expand-on-click
        :default-expanded-keys="defaultExpandedKeys"
        key-field="key"
        children-field="children"
        default-expand-all
        virtual-scroll
        style="height: 100%"
        class="file-tree"
      />
    </div>
  </div>
</template>

<script setup lang="ts">
import type { Torrent, TorrentSetArgs } from '@/api/rpc'
import { rpc } from '@/api/rpc'
import { formatSize } from '@/utils'
import { getPriorityString, type PriorityNumberType } from '@/types/tr'
import type { TreeOption } from 'naive-ui'
import { h, computed, ref, watch } from 'vue'
import { NProgress, NTag, NDropdown, NButton, useMessage, NText, NTree } from 'naive-ui'
import { FolderOutline, DocumentOutline } from '@vicons/ionicons5'
import { priorityOptions, priorityTagColorConfig } from '@/components/AppHeader/priority'
import { useTorrentStore } from '@/store'
import { useI18n } from 'vue-i18n'

const { t } = useI18n()

interface FileTreeNode extends TreeOption {
  key: string
  label: string
  fileIndex?: number
  isDirectory: boolean
  size: number
  progress: number
  priority: number
  wanted: boolean
  children?: FileTreeNode[]
  path: string
}

const props = defineProps<{ torrent: Torrent }>()
const message = useMessage()
const treeRef = ref()
const torrentStore = useTorrentStore()

// 响应式数据
const checkedKeys = ref<string[]>([])

// 构建文件树结构
const treeData = computed(() => {
  if (!props.torrent.files || !props.torrent.fileStats) {
    return []
  }

  const files = props.torrent.files
  const fileStats = props.torrent.fileStats
  const root: FileTreeNode[] = []
  const pathMap = new Map<string, FileTreeNode>()

  files.forEach((file, index) => {
    const stat = fileStats[index]
    if (!stat) {
      return
    }

    const parts = file.name.split('/')
    let currentPath = ''
    let currentLevel = root

    // 构建目录结构
    for (let i = 0; i < parts.length; i++) {
      const part = parts[i]
      const isLastPart = i === parts.length - 1
      currentPath = currentPath ? `${currentPath}/${part}` : part

      let existingNode = pathMap.get(currentPath)

      if (!existingNode) {
        const node: FileTreeNode = {
          key: currentPath,
          label: part,
          isDirectory: !isLastPart,
          size: isLastPart ? file.length : 0,
          progress: isLastPart ? (file.length > 0 ? file.bytesCompleted / file.length : 0) : 0,
          priority: isLastPart ? stat.priority : 0,
          wanted: isLastPart ? stat.wanted : true,
          children: isLastPart ? undefined : [],
          path: currentPath,
          fileIndex: isLastPart ? index : undefined
        }

        pathMap.set(currentPath, node)
        currentLevel.push(node)
        existingNode = node
      }

      if (!isLastPart && existingNode.children) {
        currentLevel = existingNode.children
      }
    }
  })

  // 更新目录的统计信息
  const updateDirectoryStats = (nodes: FileTreeNode[]) => {
    nodes.forEach((node) => {
      if (node.isDirectory && node.children) {
        updateDirectoryStats(node.children)

        // 计算目录的总大小和进度
        let totalSize = 0
        let totalCompleted = 0
        let allWanted = true
        let hasWanted = false

        node.children.forEach((child) => {
          totalSize += child.size
          totalCompleted += child.size * child.progress
          if (child.wanted) {
            hasWanted = true
          } else {
            allWanted = false
          }
        })

        node.size = totalSize
        node.progress = totalSize > 0 ? totalCompleted / totalSize : 0
        node.wanted = allWanted || hasWanted // 如果有子文件需要下载，则目录也标记为需要
      }
    })
  }

  updateDirectoryStats(root)
  return root
})

// 计算统计信息
const totalFiles = computed(() => props.torrent.files?.length || 0)
const selectedFiles = computed(() => {
  return props.torrent.fileStats?.filter((stat) => stat.wanted).length || 0
})
const totalSize = computed(() => {
  return props.torrent.files?.reduce((sum, file) => sum + file.length, 0) || 0
})

// 默认展开的节点
const defaultExpandedKeys = computed(() => {
  const keys: string[] = []
  const collectDirectoryKeys = (nodes: FileTreeNode[]) => {
    nodes.forEach((node) => {
      if (node.isDirectory) {
        keys.push(node.key)
        if (node.children) {
          collectDirectoryKeys(node.children)
        }
      }
    })
  }
  collectDirectoryKeys(treeData.value)
  return keys
})

// 勾选仅作为批量操作的选择集，不与服务端 wanted 状态联动；切换种子时清空
watch(
  () => props.torrent.id,
  () => {
    checkedKeys.value = []
  }
)

// 渲染文件/目录标签
const renderLabel = (props: any) => {
  const option = props.option as FileTreeNode
  return h('span', { class: 'truncate flex-1' }, option.label)
}

// 渲染前缀图标
const renderPrefix = (props: any) => {
  const option = props.option as FileTreeNode
  const IconComponent = option.isDirectory ? FolderOutline : DocumentOutline
  return h('div', { class: 'flex items-center' }, [h(IconComponent, { size: 16, class: 'w-[16px] h-[16px]' })])
}

// 渲染后缀信息
const renderSuffix = (props: any) => {
  const option = props.option as FileTreeNode
  return h('div', { class: 'flex items-center gap-1 text-xs' }, [
    // 大小
    h('span', { class: 'text-gray-500 min-w-[80px] text-right' }, formatSize(option.size)),
    // 进度条
    h(NProgress, {
      type: 'line',
      size: 'small',
      percentage: Math.round(option.progress * 100),
      'show-indicator': false,
      style: { width: '100px' }
    }),
    // 进度百分比
    h('span', { class: 'text-gray-500 min-w-[40px] text-right' }, `${Math.round(option.progress * 100)}%`),
    // 优先级下拉选择（未勾选下载的文件显示“不下载”标签）
    !option.isDirectory
      ? h(
          NDropdown,
          {
            options: priorityOptions,
            size: 'small',
            trigger: 'click',
            onSelect: (key: number) => handleFilePriority(option.fileIndex!, key)
          },
          () => {
            const isSkipped = !option.wanted
            const tagColor = isSkipped ? priorityTagColorConfig['-2'] : priorityTagColorConfig[option.priority]
            const tagText = isSkipped
              ? t('priority.skip')
              : getPriorityString(option.priority as PriorityNumberType) || t('torrentDetail.files.normal')
            return h(
              NTag,
              {
                size: 'small',
                color: tagColor,
                round: true,
                bordered: false,
                style: { cursor: 'pointer', minWidth: '48px', justifyContent: 'center' }
              },
              () => tagText
            )
          }
        )
      : null
  ])
}

// 处理选中状态变化：仅更新本地选择集，不触发 torrent-set；
// 只有「批量设置优先级」和「单文件状态标签修改」才会提交服务端
const onCheckedKeysChange = (keys: string[]) => {
  checkedKeys.value = keys
}

// 全选：仅更新本地选择集
const selectAll = () => {
  checkedKeys.value = props.torrent.files?.map((file) => file.name) || []
}

// 批量设置优先级
const handleBatchPriority = async (priority: number) => {
  const selectedIndices: number[] = []

  props.torrent.files?.forEach((file, index) => {
    if (checkedKeys.value.includes(file.name)) {
      selectedIndices.push(index)
    }
  })

  if (selectedIndices.length === 0) {
    message.warning(t('torrentDetail.files.pleaseSelectFiles'))
    return
  }

  try {
    const args: TorrentSetArgs = { ids: props.torrent.id }

    if (priority === -2) {
      // 不下载：移出下载列表
      args['files-unwanted'] = selectedIndices
    } else {
      // 设置优先级前先确保文件在下载列表中
      args['files-wanted'] = selectedIndices
      const priorityArgs: Record<string, number[]> = {}
      if (priority === 1) {
        priorityArgs['priority-high'] = selectedIndices
      } else if (priority === -1) {
        priorityArgs['priority-low'] = selectedIndices
      } else {
        priorityArgs['priority-normal'] = selectedIndices
      }
      Object.assign(args, priorityArgs)
    }

    await rpc.torrentSet(args)
    torrentStore.fetchDetails()
    // 批量设置完成后重置所有文件的选中状态
    checkedKeys.value = []
    message.success(
      t('torrentDetail.files.prioritySet', { count: selectedIndices.length, priority: getPriorityString(priority as PriorityNumberType) })
    )
  } catch (error) {
    console.error('设置优先级失败:', error)
    message.error(t('torrentDetail.files.setPriorityFailed'))
  }
}

// 单个文件优先级设置
const handleFilePriority = async (fileIndex: number, priority: number) => {
  try {
    const args: TorrentSetArgs = { ids: props.torrent.id }

    if (priority === -2) {
      // 不下载：移出下载列表
      args['files-unwanted'] = [fileIndex]
    } else {
      // 设置优先级前先确保文件在下载列表中
      args['files-wanted'] = [fileIndex]
      const priorityArgs: Record<string, number[]> = {}
      if (priority === 1) {
        priorityArgs['priority-high'] = [fileIndex]
      } else if (priority === -1) {
        priorityArgs['priority-low'] = [fileIndex]
      } else {
        priorityArgs['priority-normal'] = [fileIndex]
      }
      Object.assign(args, priorityArgs)
    }

    await rpc.torrentSet(args)
    torrentStore.fetchDetails()

    message.success(t('torrentDetail.files.filePrioritySet', { priority: getPriorityString(priority as PriorityNumberType) }))
  } catch (error) {
    console.error('设置文件优先级失败:', error)
    message.error(t('torrentDetail.files.setFilePriorityFailed'))
  }
}
</script>

<style scoped lang="less">
.files-tab {
  overflow: hidden;
  height: 100%;
  .title {
    flex-grow: 0;
    padding: 4px 16px;
    display: flex;
    width: 100%;
    justify-content: space-between;
  }
  .content {
    position: absolute;
    width: 100%;
    height: calc(100% - 36px);
    left: 0;
    top: 36px;
  }
  :deep(.n-tree) {
    .n-tree-node-content {
      height: 32px;

      .n-tree-node-content__text {
        flex: 1;
        display: flex;
        align-items: center;
        justify-content: space-between;
        min-width: 0;
      }
    }

    .n-tree-node-switcher {
      display: flex;
      align-items: center;
      justify-content: center;
    }
  }
}

.file-tree {
  :deep(.n-tree-node-wrapper) {
    .n-tree-node-content {
      padding-right: 8px;
    }
  }
}
</style>
