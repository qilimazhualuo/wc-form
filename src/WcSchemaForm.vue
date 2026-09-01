<script setup lang="ts">
import { computed, reactive, watch } from 'vue'
import type { PageField } from 'wc-core'
import { isAuditColumn } from 'wc-core'

const props = withDefaults(defineProps<{
    fields: PageField[]
    modelValue?: Record<string, unknown>
    disabled?: boolean
    hideLockedAndGeom?: boolean
    editableGeom?: boolean
    hideAuditFields?: boolean
}>(), {
    modelValue: () => ({}),
    disabled: false,
    hideLockedAndGeom: true,
    editableGeom: false,
    hideAuditFields: false,
})

const emit = defineEmits<{
    'update:modelValue': [value: Record<string, unknown>]
    change: [value: Record<string, unknown>]
}>()

const formState = reactive<Record<string, unknown>>({})

const isGeomField = (fieldItem: PageField) =>
    fieldItem.inputType === 'geom' || fieldItem.lockReason === 'geometry'

const GEOM_PLACEHOLDER = '请输入 GeoJSON Geometry，例如 Point: { type, coordinates: [lng, lat] }'

const resolvePlaceholder = (fieldItem: PageField) => {
    if (isGeomField(fieldItem)) {
        return GEOM_PLACEHOLDER
    }
    return fieldItem.comment || fieldItem.title
}

const isFieldReadonly = (fieldItem: PageField) =>
    !!props.disabled || !!fieldItem.disabled

const isFieldEditable = (fieldItem: PageField) => !isFieldReadonly(fieldItem)

const visibleFields = computed(() => {
    const ordered = [...props.fields]
    return ordered.filter((fieldItem) => {
        if (isGeomField(fieldItem)) {
            return !!props.editableGeom
        }
        if (props.hideLockedAndGeom) {
            if (fieldItem.lockReason === 'pk') {
                return false
            }
            if (
                fieldItem.locked
                && fieldItem.lockReason !== 'audit'
                && !isAuditColumn(fieldItem.field)
            ) {
                return false
            }
        }
        if (props.hideAuditFields && isAuditColumn(fieldItem.field)) {
            return false
        }
        return true
    })
})

const syncFromModel = (source: Record<string, unknown>) => {
    Object.keys(formState).forEach((key) => {
        delete formState[key]
    })
    visibleFields.value.forEach((fieldItem) => {
        const fieldName = fieldItem.field
        const rawValue = source[fieldName]
        if (isGeomField(fieldItem)) {
            if (rawValue && typeof rawValue === 'object') {
                formState[fieldName] = JSON.stringify(rawValue, null, 2)
                return
            }
            formState[fieldName] = rawValue ?? ''
            return
        }
        if (fieldItem.inputType === 'switch' || fieldItem.inputType === 'checkbox') {
            formState[fieldName] = Boolean(rawValue)
            return
        }
        if (fieldItem.inputType === 'number') {
            formState[fieldName] = rawValue === null || rawValue === undefined || rawValue === ''
                ? null
                : Number(rawValue)
            return
        }
        formState[fieldName] = rawValue ?? ''
    })
}

watch(
    () => [props.modelValue, props.fields] as const,
    () => {
        syncFromModel(props.modelValue || {})
    },
    { immediate: true, deep: true },
)

const collectEditableValues = () => {
    const nextValue: Record<string, unknown> = {}
    visibleFields.value.forEach((fieldItem) => {
        if (!isFieldEditable(fieldItem)) {
            return
        }
        nextValue[fieldItem.field] = formState[fieldItem.field]
    })
    return nextValue
}

const emitValue = () => {
    const nextValue: Record<string, unknown> = {
        ...(props.modelValue || {}),
        ...collectEditableValues(),
    }
    visibleFields.value.forEach((fieldItem) => {
        if (isAuditColumn(fieldItem.field)) {
            delete nextValue[fieldItem.field]
        }
    })
    emit('update:modelValue', nextValue)
    emit('change', nextValue)
}

const handleFieldChange = () => {
    emitValue()
}

const getSwitchChecked = (fieldName: string): boolean =>
    Boolean(formState[fieldName])

const handleSwitchChange = (fieldName: string, checked: unknown) => {
    formState[fieldName] = Boolean(checked)
    handleFieldChange()
}

const toSelectOptionValue = (value: string | number | boolean): string | number => {
    if (typeof value === 'boolean') {
        return String(value)
    }
    return value
}

const toSelectOptions = (fieldItem: PageField) =>
    (fieldItem.options || []).map((optionItem) => ({
        label: optionItem.label,
        value: toSelectOptionValue(optionItem.value),
    }))

defineExpose({
    getValues: () => collectEditableValues(),
})
</script>

<template>
    <div class="wc-page-schema-form">
        <div
            v-for="fieldItem in visibleFields"
            :key="fieldItem.key || fieldItem.field"
            class="wc-page-schema-form__item"
        >
            <label class="wc-page-schema-form__label">
                <span
                    v-if="fieldItem.required"
                    class="wc-page-schema-form__required"
                >*</span>
                {{ fieldItem.title || fieldItem.field }}
                <span
                    v-if="fieldItem.disabled"
                    class="wc-page-schema-form__readonly-tag"
                >
                    只读
                </span>
            </label>
            <div class="wc-page-schema-form__control">
                <slot
                    :name="`field-control-${fieldItem.field}`"
                    :field="fieldItem"
                    :disabled="disabled || isFieldReadonly(fieldItem)"
                >
                    <a-textarea
                        v-if="fieldItem.inputType === 'textarea' || isGeomField(fieldItem)"
                        v-model:value="formState[fieldItem.field]"
                        :disabled="disabled || isFieldReadonly(fieldItem)"
                        :rows="isGeomField(fieldItem) ? 8 : 3"
                        :placeholder="resolvePlaceholder(fieldItem)"
                        @change="handleFieldChange"
                    />
                    <a-input-number
                        v-else-if="fieldItem.inputType === 'number'"
                        v-model:value="formState[fieldItem.field]"
                        :disabled="disabled || isFieldReadonly(fieldItem)"
                        style="width: 100%"
                        @change="handleFieldChange"
                    />
                    <a-select
                        v-else-if="fieldItem.inputType === 'select' || fieldItem.inputType === 'radio'"
                        v-model:value="formState[fieldItem.field]"
                        :disabled="disabled || isFieldReadonly(fieldItem)"
                        :options="toSelectOptions(fieldItem)"
                        allow-clear
                        style="width: 100%"
                        @change="handleFieldChange"
                    />
                    <a-switch
                        v-else-if="fieldItem.inputType === 'switch' || fieldItem.inputType === 'checkbox'"
                        :checked="getSwitchChecked(fieldItem.field)"
                        :disabled="disabled || isFieldReadonly(fieldItem)"
                        @update:checked="(checked) => handleSwitchChange(fieldItem.field, checked)"
                    />
                    <a-input
                        v-else-if="fieldItem.inputType === 'date' || fieldItem.inputType === 'datetime'"
                        v-model:value="formState[fieldItem.field]"
                        :disabled="disabled || isFieldReadonly(fieldItem)"
                        :placeholder="fieldItem.inputType === 'datetime' ? 'YYYY-MM-DD HH:mm:ss' : 'YYYY-MM-DD'"
                        @change="handleFieldChange"
                    />
                    <a-input
                        v-else
                        v-model:value="formState[fieldItem.field]"
                        :disabled="disabled || isFieldReadonly(fieldItem)"
                        :placeholder="fieldItem.comment || fieldItem.title"
                        @change="handleFieldChange"
                    />
                </slot>
            </div>
        </div>
        <a-empty
            v-if="!visibleFields.length"
            description="无可编辑字段"
        />
    </div>
</template>

<style lang="less" scoped>
.wc-page-schema-form {
    display: flex;
    flex-direction: column;
    gap: 12px;

    &__item {
        display: flex;
        flex-direction: column;
        gap: 6px;
    }

    &__label {
        font-size: 13px;
        color: rgba(0, 0, 0, 0.75);
        line-height: 1.4;
    }

    &__required {
        color: #ff4d4f;
        margin-right: 4px;
    }

    &__readonly-tag {
        margin-left: 6px;
        font-size: 12px;
        color: rgba(0, 0, 0, 0.45);
    }

    &__control {
        width: 100%;
    }
}
</style>
