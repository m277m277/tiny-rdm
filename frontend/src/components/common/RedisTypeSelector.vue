<script setup>
import { computed, h } from 'vue'
import { NSpace, useThemeVars } from 'naive-ui'
import { types, typesBgColor, typesColor, typesShortName } from '@/consts/support_redis_type.js'
import { get, isEmpty, map, toUpper } from 'lodash'
import RedisTypeTag from '@/components/common/RedisTypeTag.vue'

const props = defineProps({
    value: {
        type: String,
        default: 'ALL',
    },
    placement: {
        type: String,
        default: 'bottom-start',
    },
    disabled: {
        type: Boolean,
        default: false,
    },
    disableTip: {
        type: String,
        default: '',
    },
})

const emit = defineEmits(['update:value', 'select'])

const options = computed(() => {
    const opts = map(types, (v) => ({
        label: v,
        key: v,
    }))
    return [{ label: 'ALL', key: 'ALL' }, ...opts]
})

const themeVars = useThemeVars()
const renderIcon = (option) => {
    return h(RedisTypeTag, {
        type: option.key,
        defaultLabel: 'A',
        short: true,
        size: 'small',
        inverse: option.key === props.value,
    })
}

const renderLabel = (option) => {
    const children = [
        h(
            'div',
            {
                style: {
                    fontWeight: option.key === props.value ? 'bold' : 'normal',
                },
            },
            option.label,
        ),
        h(
            'div',
            { style: { width: '16px' } },
            h(RedisTypeTag, {
                type: toUpper(option.key),
                point: true,
                style: { display: option.key === props.value ? 'block' : 'none' },
            }),
        ),
    ]
    return h(NSpace, { align: 'center', wrapItem: false }, () => children)
}

const fontColor = computed(() => {
    return get(typesColor, props.value, '')
})

const backgroundColor = computed(() => {
    return get(typesBgColor, props.value, '')
})

const displayValue = computed(() => {
    return get(typesShortName, toUpper(props.value), 'A')
})

const handleSelect = (select) => {
    if (props.value !== select) {
        emit('update:value', select)
        emit('select', select)
    }
}
</script>

<template>
    <template v-if="props.disabled">
        <n-tooltip :disabled="isEmpty(props.disableTip)">
            <div>{{ props.disableTip }}</div>
            <template #trigger>
                <n-tag
                    :bordered="true"
                    :color="{ color: backgroundColor, textColor: fontColor }"
                    class="redis-tag"
                    disabled
                    size="medium"
                    strong>
                    {{ displayValue }}
                </n-tag>
            </template>
        </n-tooltip>
    </template>
    <template v-else>
        <n-dropdown
            :disabled="props.disabled"
            :options="options"
            :placement="props.placement"
            :render-icon="renderIcon"
            :render-label="renderLabel"
            show-arrow
            @select="handleSelect">
            <n-tag
                :bordered="true"
                :color="{ color: backgroundColor, textColor: fontColor }"
                :disabled="props.disabled"
                class="redis-tag"
                size="medium"
                strong>
                {{ displayValue }}
            </n-tag>
        </n-dropdown>
    </template>
</template>

<style lang="scss" scoped>
.redis-tag {
    padding: 0 10px;
}

:deep(.dropdown-type-item) {
    padding: 10px;
}
</style>
