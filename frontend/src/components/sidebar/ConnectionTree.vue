<script setup>
import useDialogStore from '../../stores/dialog.js'
import { h, nextTick, onMounted, reactive, ref } from 'vue'
import useConnectionStore from '../../stores/connections.js'
import { NIcon, useMessage } from 'naive-ui'
import { ConnectionType } from '../../consts/connection_type.js'
import ToggleFolder from '../icons/ToggleFolder.vue'
import ToggleServer from '../icons/ToggleServer.vue'
import { indexOf } from 'lodash'
import Config from '../icons/Config.vue'
import Delete from '../icons/Delete.vue'
import Refresh from '../icons/Refresh.vue'
import Unlink from '../icons/Unlink.vue'
import CopyLink from '../icons/CopyLink.vue'
import Connect from '../icons/Connect.vue'
import { useI18n } from 'vue-i18n'
import useTabStore from '../../stores/tab.js'
import Edit from '../icons/Edit.vue'

const i18n = useI18n()
const loadingConnection = ref(false)
const openingConnection = ref(false)
const connectionStore = useConnectionStore()
const tabStore = useTabStore()
const dialogStore = useDialogStore()
const message = useMessage()

const expandedKeys = ref([])

onMounted(async () => {
    try {
        loadingConnection.value = true
        await nextTick()
        await connectionStore.initConnections()
    } finally {
        loadingConnection.value = false
    }
})

const contextMenuParam = reactive({
    show: false,
    x: 0,
    y: 0,
    options: null,
    currentNode: null,
})

const renderIcon = (icon) => {
    return () => {
        return h(NIcon, null, {
            default: () => h(icon),
        })
    }
}
const menuOptions = {
    [ConnectionType.Group]: ({ opened }) => [
        {
            key: 'group_reload',
            label: i18n.t('edit_conn_group'),
            icon: renderIcon(Config),
        },
        {
            key: 'group_delete',
            label: i18n.t('remove_conn_group'),
            icon: renderIcon(Delete),
        },
    ],
    [ConnectionType.Server]: ({ connected }) => {
        if (connected) {
            return [
                {
                    key: 'server_reload',
                    label: i18n.t('reload'),
                    icon: renderIcon(Refresh),
                },
                {
                    key: 'server_close',
                    label: i18n.t('disconnect'),
                    icon: renderIcon(Unlink),
                },
                {
                    key: 'server_dup',
                    label: i18n.t('dup_conn'),
                    icon: renderIcon(CopyLink),
                },
                {
                    key: 'server_config',
                    label: i18n.t('edit_conn'),
                    icon: renderIcon(Config),
                },
                {
                    type: 'divider',
                    key: 'd1',
                },
                {
                    key: 'server_remove',
                    label: i18n.t('remove_conn'),
                    icon: renderIcon(Delete),
                },
            ]
        } else {
            return [
                {
                    key: 'server_open',
                    label: i18n.t('open_connection'),
                    icon: renderIcon(Connect),
                },
                {
                    key: 'server_edit',
                    label: i18n.t('edit_conn'),
                    icon: renderIcon(Edit),
                },
                {
                    type: 'divider',
                    key: 'd1',
                },
                {
                    key: 'server_delete',
                    label: i18n.t('remove_conn'),
                    icon: renderIcon(Delete),
                },
            ]
        }
    },
}

const renderLabel = ({ option }) => {
    // switch (option.type) {
    //     case ConnectionType.Server:
    //         return h(ConnectionTreeItem, { title: option.label })
    // }
    return option.label
}

const renderPrefix = ({ option }) => {
    switch (option.type) {
        case ConnectionType.Group:
            const opened = indexOf(expandedKeys.value, option.key) !== -1
            return h(
                NIcon,
                { size: 20 },
                {
                    default: () => h(ToggleFolder, { modelValue: opened }),
                }
            )
        case ConnectionType.Server:
            const connected = connectionStore.isConnected(option.name)
            return h(
                NIcon,
                { size: 20 },
                {
                    default: () => h(ToggleServer, { modelValue: !!connected }),
                }
            )
    }
}

const onUpdateExpandedKeys = (value, option, meta) => {
    expandedKeys.value = value
}

/**
 * Open connection
 * @param name
 * @returns {Promise<void>}
 */
const openConnection = async (name) => {
    try {
        openingConnection.value = true
        await connectionStore.openConnection(name)
        tabStore.upsertTab({
            server: name,
        })
    } catch (e) {
        message.error(e.message)
        // node.isLeaf = undefined
    } finally {
        openingConnection.value = false
    }
}

const nodeProps = ({ option }) => {
    return {
        onDblclick: async () => {
            if (option.type === ConnectionType.Server) {
                openConnection(option.name).then(() => {})
            }
        },
        onContextmenu(e) {
            e.preventDefault()
            const mop = menuOptions[option.type]
            if (mop == null) {
                return
            }
            contextMenuParam.show = false
            nextTick().then(() => {
                contextMenuParam.options = mop(option)
                contextMenuParam.currentNode = option
                contextMenuParam.x = e.clientX
                contextMenuParam.y = e.clientY
                contextMenuParam.show = true
            })
        },
    }
}

const renderContextLabel = (option) => {
    return h('div', { class: 'context-menu-item' }, option.label)
}

const handleSelectContextMenu = (key) => {
    contextMenuParam.show = false
    const { name, db, key: nodeKey, redisKey } = contextMenuParam.currentNode
    switch (key) {
        case 'server_open':
            openConnection(name).then(() => {})
    }
    console.warn('TODO: handle context menu:' + key)
}
</script>

<template>
    <n-tree
        :animated="false"
        :block-line="true"
        :block-node="true"
        :cancelable="false"
        :data="connectionStore.connections"
        :expand-on-click="true"
        :expanded-keys="expandedKeys"
        :node-props="nodeProps"
        :on-update:expanded-keys="onUpdateExpandedKeys"
        :render-label="renderLabel"
        :render-prefix="renderPrefix"
        class="fill-height"
        virtual-scroll
    />

    <!-- status display modal -->
    <n-modal :show="loadingConnection || openingConnection" transform-origin="center">
        <n-card
            :bordered="false"
            :content-style="{ textAlign: 'center' }"
            aria-model="true"
            role="dialog"
            style="width: 400px"
        >
            <n-spin>
                <template #description>
                    {{ openingConnection ? $t('opening_connection') : '' }}
                </template>
            </n-spin>
        </n-card>
    </n-modal>

    <!-- context menu -->
    <n-dropdown
        :animated="false"
        :options="contextMenuParam.options"
        :render-label="renderContextLabel"
        :show="contextMenuParam.show"
        :x="contextMenuParam.x"
        :y="contextMenuParam.y"
        placement="bottom-start"
        trigger="manual"
        @clickoutside="contextMenuParam.show = false"
        @select="handleSelectContextMenu"
    />
</template>

<style lang="scss" scoped></style>