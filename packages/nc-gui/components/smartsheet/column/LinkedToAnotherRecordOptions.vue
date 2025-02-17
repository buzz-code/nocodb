<script setup lang="ts">
import { ModelTypes, MssqlUi, RelationTypes, SqliteUi, UITypes, ViewTypes } from 'nocodb-sdk'

const props = defineProps<{
  value: any
  isEdit: boolean
}>()

const emit = defineEmits(['update:value'])

const vModel = useVModel(props, 'value', emit)

const isEdit = toRef(props, 'isEdit')

const meta = inject(MetaInj, ref())

const filterRef = ref()

const { setAdditionalValidations, setPostSaveOrUpdateCbk, validateInfos, onDataTypeChange, sqlUi, isXcdbBase } =
  useColumnCreateStoreOrThrow()

const baseStore = useBase()
const { tables } = storeToRefs(baseStore)

const viewsStore = useViewsStore()
const { viewsByTable } = storeToRefs(viewsStore)

const { t } = useI18n()

if (!isEdit.value) {
  setAdditionalValidations({
    childId: [{ required: true, message: t('general.required') }],
  })
}

const onUpdateDeleteOptions = sqlUi === MssqlUi ? ['NO ACTION'] : ['NO ACTION', 'CASCADE', 'RESTRICT', 'SET NULL', 'SET DEFAULT']

if (!isEdit.value) {
  if (!vModel.value.parentId) vModel.value.parentId = meta.value?.id
  if (!vModel.value.childColumn) vModel.value.childColumn = `${meta.value?.table_name}_id`
  if (!vModel.value.childTable) vModel.value.childTable = meta.value?.table_name
  if (!vModel.value.parentTable) vModel.value.parentTable = vModel.value.rtn || ''
  if (!vModel.value.parentColumn) vModel.value.parentColumn = vModel.value.rcn || ''

  if (!vModel.value.type) vModel.value.type = 'mm'
  if (!vModel.value.onUpdate) vModel.value.onUpdate = onUpdateDeleteOptions[0]
  if (!vModel.value.onDelete) vModel.value.onDelete = onUpdateDeleteOptions[0]
  if (!vModel.value.virtual) vModel.value.virtual = sqlUi === SqliteUi // appInfo.isCloud || sqlUi === SqliteUi
  if (!vModel.value.alias) vModel.value.alias = vModel.value.column_name
}
if (!vModel.value.childId) vModel.value.childId = vModel.value?.colOptions?.fk_related_model_id || null
if (!vModel.value.childViewId) vModel.value.childViewId = vModel.value?.colOptions?.fk_target_view_id || null

const advancedOptions = ref(false)

const refTables = computed(() => {
  if (!tables.value || !tables.value.length) {
    return []
  }

  return tables.value.filter((t) => t.type === ModelTypes.TABLE && t.source_id === meta.value?.source_id)
})

const refViews = computed(() => {
  if (!vModel.value.childId) return []
  const views = viewsByTable.value.get(vModel.value.childId)

  return (views || []).filter((v) => v.type !== ViewTypes.FORM)
})

const filterOption = (value: string, option: { key: string }) => option.key.toLowerCase().includes(value.toLowerCase())

const isLinks = computed(() => vModel.value.uidt === UITypes.Links && vModel.value.type !== RelationTypes.ONE_TO_ONE)

const { metas, getMeta } = useMetas()

watch(
  () => vModel.value.childId,
  async (tableId) => {
    if (tableId) {
      getMeta(tableId).catch(() => {
        // ignore
      })
      viewsStore
        .loadViews({
          ignoreLoading: true,
          tableId,
        })
        .catch(() => {
          // ignore
        })
    }
  },
  {
    immediate: true,
  },
)

vModel.value.meta = vModel.value.meta || {}
const limitRecToView = ref(!!vModel.value.childViewId)
const limitRecToCond = computed({
  get() {
    return !!vModel.value.meta?.enableConditions
  },
  set(value) {
    vModel.value.meta = vModel.value.meta || {}
    vModel.value.meta.enableConditions = value
  },
})

const onLimitRecToViewChange = (value: boolean) => {
  if (!value) {
    vModel.value.childViewId = null
  }
}

provide(
  MetaInj,
  computed(() => metas.value[vModel.value.childId] || {}),
)

onMounted(() => {
  setPostSaveOrUpdateCbk(async ({ colId, column }) => {
    await filterRef.value?.applyChanges(colId || column.id, false)
  })
})

onUnmounted(() => {
  setPostSaveOrUpdateCbk(null)
})

const referenceTableChildId = computed({
  get: () => (isEdit.value ? vModel.value?.colOptions?.fk_related_model_id : vModel.value?.childId) ?? null,
  set: (value) => {
    if (!isEdit.value && value) {
      vModel.value.childId = value
    }
  },
})

const linkType = computed({
  get: () => (isEdit.value ? vModel.value?.colOptions?.type : vModel.value?.type) ?? null,
  set: (value) => {
    if (!isEdit.value && value) {
      vModel.value.type = value
    }
  },
})
</script>

<template>
  <div class="w-full flex flex-col gap-4">
    <div class="flex flex-col gap-4">
      <a-form-item :label="$t('labels.relationType')" v-bind="validateInfos.type" class="nc-ltar-relation-type">
        <a-radio-group v-model:value="linkType" name="type" v-bind="validateInfos.type" :disabled="isEdit">
          <a-radio value="mm" data-testid="Many to Many">
            <span class="nc-ltar-icon nc-mm-icon">
              <GeneralIcon icon="mm_solid" />
            </span>
            {{ $t('title.manyToMany') }}
          </a-radio>
          <a-radio value="hm" data-testid="Has Many">
            <span class="nc-ltar-icon nc-hm-icon">
              <GeneralIcon icon="hm_solid" />
            </span>
            {{ $t('title.hasMany') }}
          </a-radio>
          <a-radio value="oo" data-testid="One to One">
            <span class="nc-ltar-icon nc-oo-icon">
              <GeneralIcon icon="oneToOneSolid" />
            </span>
            {{ $t('title.oneToOne') }}
          </a-radio>
        </a-radio-group>
      </a-form-item>

      <a-form-item class="flex w-full nc-ltar-child-table" v-bind="validateInfos.childId">
        <a-select
          v-model:value="referenceTableChildId"
          show-search
          :disabled="isEdit"
          :filter-option="filterOption"
          placeholder="select table to link"
          dropdown-class-name="nc-dropdown-ltar-child-table"
          @change="onDataTypeChange"
        >
          <template #suffixIcon>
            <GeneralIcon icon="arrowDown" class="text-gray-700" />
          </template>
          <a-select-option v-for="table of refTables" :key="table.title" :value="table.id">
            <div class="flex w-full items-center gap-2">
              <div class="min-w-5 flex items-center justify-center">
                <GeneralTableIcon :meta="table" class="text-gray-500" />
              </div>
              <NcTooltip class="flex-1 truncate" show-on-truncate-only>
                <template #title>{{ table.title }}</template>
                <span>{{ table.title }}</span>
              </NcTooltip>
            </div>
          </a-select-option>
        </a-select>
      </a-form-item>

      <div v-if="isEeUI" class="w-full flex-col">
        <div class="flex gap-2 items-center" :class="{ 'mb-2': limitRecToView }">
          <a-switch
            v-model:checked="limitRecToView"
            v-e="['c:link:limit-record-by-view', { status: limitRecToView }]"
            size="small"
            :disabled="!vModel.childId"
            @change="onLimitRecToViewChange"
          ></a-switch>
          <span
            v-e="['c:link:limit-record-by-view', { status: limitRecToView }]"
            class="text-s"
            data-testid="nc-limit-record-view"
            @click="limitRecToView = !!vModel.childId && !limitRecToView"
            >Limit record selection to a view</span
          >
        </div>
        <a-form-item v-if="limitRecToView" class="!pl-8 flex w-full pb-2 mt-4 space-y-2 nc-ltar-child-view">
          <NcSelect
            v-model:value="vModel.childViewId"
            :placeholder="$t('labels.selectView')"
            show-search
            :filter-option="filterOption"
            dropdown-class-name="nc-dropdown-ltar-child-view"
          >
            <a-select-option v-for="view of refViews" :key="view.title" :value="view.id">
              <div class="flex w-full items-center gap-2">
                <div class="min-w-5 flex items-center justify-center">
                  <GeneralViewIcon :meta="view" class="text-gray-500" />
                </div>
                <NcTooltip class="flex-1 truncate" show-on-truncate-only>
                  <template #title>{{ view.title }}</template>
                  <span>{{ view.title }}</span>
                </NcTooltip>
              </div>
            </a-select-option>
          </NcSelect>
        </a-form-item>

        <div class="mt-4 flex gap-2 items-center" :class="{ 'mb-2': limitRecToCond }">
          <a-switch
            v-model:checked="limitRecToCond"
            v-e="['c:link:limit-record-by-filter', { status: limitRecToCond }]"
            :disabled="!vModel.childId"
            size="small"
          ></a-switch>
          <span
            v-e="['c:link:limit-record-by-filter', { status: limitRecToCond }]"
            data-testid="nc-limit-record-filters"
            @click="limitRecToCond = !!vModel.childId && !limitRecToCond"
          >
            Limit record selection to filters
          </span>
        </div>
        <div v-if="limitRecToCond" class="overflow-auto">
          <LazySmartsheetToolbarColumnFilter
            ref="filterRef"
            v-model="vModel.filters"
            class="!pl-8 !p-0 max-w-620px"
            :auto-save="false"
            :show-loading="false"
            :link="true"
            :root-meta="meta"
            :link-col-id="vModel.id"
          />
        </div>
      </div>
    </div>
    <template v-if="(!isXcdbBase && !isEdit) || isLinks">
      <div>
        <NcButton
          size="small"
          type="text"
          class="!text-gray-500 !hover:text-gray-700"
          @click.stop="advancedOptions = !advancedOptions"
        >
          <div class="flex items-center gap-2">
            <span class="first-letter:capitalize">
              {{ $t('title.advancedSettings').toLowerCase() }}
            </span>

            <GeneralIcon :icon="advancedOptions ? 'arrowUp' : 'arrowDown'" class="h-4 w-4" />
          </div>
        </NcButton>
      </div>

      <div v-if="advancedOptions" class="flex flex-col gap-4">
        <LazySmartsheetColumnLinkOptions v-if="isLinks" v-model:value="vModel" />
        <template v-if="!isXcdbBase && !isEdit">
          <div class="flex flex-row space-x-2">
            <a-form-item class="flex w-1/2" :label="$t('labels.onUpdate')">
              <a-select
                v-model:value="vModel.onUpdate"
                :disabled="vModel.virtual"
                name="onUpdate"
                dropdown-class-name="nc-dropdown-on-update"
                @change="onDataTypeChange"
              >
                <template #suffixIcon>
                  <GeneralIcon icon="arrowDown" class="text-gray-700" />
                </template>
                <a-select-option v-for="(option, i) of onUpdateDeleteOptions" :key="i" :value="option">
                  <template v-if="option === 'NO ACTION'">{{ $t('title.links.noAction') }}</template>
                  <template v-else-if="option === 'CASCADE'">{{ $t('title.links.cascade') }}</template>
                  <template v-else-if="option === 'RESTRICT'">{{ $t('title.links.restrict') }}</template>
                  <template v-else-if="option === 'SET NULL'">{{ $t('title.links.setNull') }}</template>
                  <template v-else-if="option === 'SET DEFAULT'">{{ $t('title.links.setDefault') }}</template>
                  <template v-else>
                    {{ option }}
                  </template>
                </a-select-option>
              </a-select>
            </a-form-item>

            <a-form-item class="flex w-1/2" :label="$t('labels.onDelete')">
              <a-select
                v-model:value="vModel.onDelete"
                :disabled="vModel.virtual"
                name="onDelete"
                dropdown-class-name="nc-dropdown-on-delete"
                @change="onDataTypeChange"
              >
                <template #suffixIcon>
                  <GeneralIcon icon="arrowDown" class="text-gray-700" />
                </template>
                <a-select-option v-for="(option, i) of onUpdateDeleteOptions" :key="i" :value="option">
                  <template v-if="option === 'NO ACTION'">{{ $t('title.links.noAction') }}</template>
                  <template v-else-if="option === 'CASCADE'">{{ $t('title.links.cascade') }}</template>
                  <template v-else-if="option === 'RESTRICT'">{{ $t('title.links.restrict') }}</template>
                  <template v-else-if="option === 'SET NULL'">{{ $t('title.links.setNull') }}</template>
                  <template v-else-if="option === 'SET DEFAULT'">{{ $t('title.links.setDefault') }}</template>
                  <template v-else>
                    {{ option }}
                  </template>
                </a-select-option>
              </a-select>
            </a-form-item>
          </div>

          <div class="flex flex-row">
            <a-form-item>
              <div class="flex items-center gap-1">
                <NcSwitch v-model:checked="vModel.virtual" @change="onDataTypeChange">
                  <div class="text-sm text-gray-800 select-none">
                    {{ $t('title.virtualRelation') }}
                  </div>
                </NcSwitch>
              </div>
            </a-form-item>
          </div>
        </template>
      </div>
    </template>
  </div>
</template>

<style lang="scss" scoped>
:deep(.nc-filter-grid) {
  @apply !pr-0;
}

:deep(.nc-ltar-relation-type .ant-radio-group) {
  @apply flex justify-between gap-2 children:(flex-1 m-0 px-2 py-1 border-1 border-gray-300 rounded-lg);

  .ant-radio-wrapper {
    @apply transition-all flex-row-reverse justify-between items-center py-1 pl-1 pr-3;

    &.ant-radio-wrapper-checked:not(.ant-radio-wrapper-disabled):focus-within {
      @apply border-brand-500;
    }

    span:not(.ant-radio):not(.nc-ltar-icon) {
      @apply flex-1 pl-0 flex items-center gap-2;
    }

    .ant-radio {
      @apply top-0;
    }

    .nc-ltar-icon {
      @apply inline-flex items-center p-1 rounded-md;
    }
  }
}

:deep(.nc-ltar-relation-type .ant-col.ant-form-item-control) {
  @apply h-8.5;
}
</style>
