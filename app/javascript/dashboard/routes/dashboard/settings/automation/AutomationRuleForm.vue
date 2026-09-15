<script setup>
import { ref, computed, h, useTemplateRef, watch } from 'vue';
import { useI18n } from 'vue-i18n';
import { useAccount } from 'dashboard/composables/useAccount';
import { useOperators } from 'dashboard/components-next/filter/operators';
import NextButton from 'dashboard/components-next/button/Button.vue';
import SidePanel from 'dashboard/components-next/side-panel/SidePanel.vue';
import {
  generateAutomationPayload,
  getActionIcon,
  getAttributes,
} from 'dashboard/helper/automationHelper';
import { getAttributeIcon } from 'dashboard/components-next/filter/helper/filterAttributeIcons';
import { provideDropdownTeleport } from 'dashboard/components-next/dropdown-menu/base/provider';
import { validateAutomation } from 'dashboard/helper/validations';
import { FEATURE_FLAGS } from 'dashboard/featureFlags';
import { DURATION_UNITS } from 'dashboard/components-next/input/constants';
import {
  AUTOMATION_RULE_EVENTS,
  AUTOMATION_ACTION_TYPES,
  SCHEDULE_ANCHOR_OPTIONS,
  SCHEDULE_DURATION_UNITS,
  DEFAULT_DELAY_MINUTES,
} from './constants';
import AutomationRunTypeSelector from './components/AutomationRunTypeSelector.vue';
import AutomationWaitCondition from './components/AutomationWaitCondition.vue';
import AutomationInstantTrigger from './components/AutomationInstantTrigger.vue';
import AutomationActions from './components/AutomationActions.vue';

const props = defineProps({
  mode: {
    type: String,
    required: true,
    validator: value => ['create', 'edit'].includes(value),
  },
  automationTypes: {
    type: Object,
    required: true,
  },
  getConditionDropdownValues: {
    type: Function,
    required: true,
  },
  getActionDropdownValues: {
    type: Function,
    required: true,
  },
  appendNewCondition: {
    type: Function,
    required: true,
  },
  appendNewAction: {
    type: Function,
    required: true,
  },
  removeFilter: {
    type: Function,
    required: true,
  },
  removeAction: {
    type: Function,
    required: true,
  },
  resetAction: {
    type: Function,
    required: true,
  },
  onEventChange: {
    type: Function,
    required: true,
  },
});

const emit = defineEmits(['save']);
const automation = defineModel('automation', { type: Object, default: null });

const INPUT_TYPE_MAP = {
  multi_select: 'multiSelect',
  search_select: 'searchSelect',
  plain_text: 'plainText',
  multi_text: 'multiText',
  date: 'date',
};

const { t } = useI18n();
const { isCloudFeatureEnabled } = useAccount();
const { operators } = useOperators();

provideDropdownTeleport();

const panelRef = ref(null);
const instantTriggerRef = useTemplateRef('instantTriggerRef');
const errors = ref({});

const eventName = computed(() => automation.value?.event_name);

// --- Schedule (time_elapsed) state ---
// We store duration as a human-friendly number + unit pair locally and
// sync the product (minutes) back into automation.schedule_duration_minutes.
const scheduleDuration = ref(1);
const scheduleUnit = ref('hours'); // key from SCHEDULE_DURATION_UNITS

const isTimerEvent = computed(() => eventName.value === 'time_elapsed');

const scheduleAnchorOptions = computed(() =>
  SCHEDULE_ANCHOR_OPTIONS.map(opt => ({
    ...opt,
    label: t(`AUTOMATION.SCHEDULE.ANCHOR.${opt.value}`),
  }))
);

const scheduleDurationUnits = computed(() =>
  SCHEDULE_DURATION_UNITS.map(unit => ({
    ...unit,
    label: t(`AUTOMATION.SCHEDULE.UNIT.${unit.value}`),
  }))
);

// Keep schedule_duration_minutes in sync whenever the human-friendly values change.
watch([scheduleDuration, scheduleUnit], ([duration, unitKey]) => {
  if (!isTimerEvent.value) return;
  const unit = SCHEDULE_DURATION_UNITS.find(u => u.key === unitKey);
  if (unit && duration > 0) {
    automation.value.schedule_duration_minutes = duration * unit.multiplier;
  }
});

// When editing an existing timer rule, decompose stored minutes back into
// a human-friendly duration + unit for display.
watch(
  () => automation.value?.schedule_duration_minutes,
  minutes => {
    if (!minutes || !isTimerEvent.value) return;
    // Pick the largest unit that divides evenly (days → hours → minutes).
    const units = [...SCHEDULE_DURATION_UNITS].reverse();
    const match = units.find(u => minutes % u.multiplier === 0);
    if (match) {
      scheduleDuration.value = minutes / match.multiplier;
      scheduleUnit.value = match.key;
    } else {
      scheduleDuration.value = minutes;
      scheduleUnit.value = 'minutes';
    }
  },
  { immediate: true }
);
// --- End schedule state ---

const isEditMode = computed(() => props.mode === 'edit');

const allowsDelayedExecution = computed(() =>
  isCloudFeatureEnabled(FEATURE_FLAGS.DELAYED_AUTOMATIONS)
);

// The wait lives here rather than in the wait section so that switching between the two run
// types doesn't discard a duration the user already typed in.
const isDelayed = ref(false);
const isSavedWait = ref(false);
const delayMinutes = ref(DEFAULT_DELAY_MINUTES);
const delayUnit = ref(DURATION_UNITS.HOURS);
// Bumped on every open() so the wait section remounts and re-reads the rule it is given.
const waitSectionKey = ref(0);

const executionDelayInvalid = computed(
  () => isDelayed.value && !Number.isFinite(delayMinutes.value)
);

const statusOptions = computed(() =>
  (props.getConditionDropdownValues('status') || [])
    .filter(option => option.id !== 'all')
    .map(option => ({ value: option.id, label: option.name }))
);

const inboxOptions = computed(
  () => props.getConditionDropdownValues('inbox_id') || []
);

// Show the wait in the largest whole unit (240 min → 4 hours). The delay is passed in by open()
// rather than read from `automation`, whose model prop only settles a tick later.
const syncDelayState = executionDelay => {
  isDelayed.value = Boolean(executionDelay);
  isSavedWait.value = isEditMode.value && Boolean(executionDelay);
  const minutes = executionDelay || DEFAULT_DELAY_MINUTES;
  if (minutes % 1440 === 0) delayUnit.value = DURATION_UNITS.DAYS;
  else if (minutes % 60 === 0) delayUnit.value = DURATION_UNITS.HOURS;
  else delayUnit.value = DURATION_UNITS.MINUTES;
  delayMinutes.value = minutes;
  waitSectionKey.value += 1;
};

watch([isDelayed, delayMinutes], () => {
  // Switching to "run instantly" hands the conditions back to the instant editor.
  if (!isDelayed.value) isSavedWait.value = false;

  if (!automation.value || !allowsDelayedExecution.value) return;
  automation.value.execution_delay = isDelayed.value
    ? delayMinutes.value
    : null;
});

const titleKey = computed(() =>
  isEditMode.value ? 'AUTOMATION.EDIT.TITLE' : 'AUTOMATION.ADD.TITLE'
);
const cancelKey = computed(() =>
  isEditMode.value
    ? 'AUTOMATION.EDIT.CANCEL_BUTTON_TEXT'
    : 'AUTOMATION.ADD.CANCEL_BUTTON_TEXT'
);
const submitKey = computed(() =>
  isEditMode.value ? 'AUTOMATION.EDIT.SUBMIT' : 'AUTOMATION.ADD.SUBMIT'
);

const getTranslatedAttributes = (type, event) => {
  return getAttributes(type, event).map(attribute => {
    const skipTranslation =
      attribute.customAttributeType ||
      ['contact_custom_attribute', 'conversation_custom_attribute'].includes(
        attribute.key
      );
    return {
      ...attribute,
      name: skipTranslation
        ? attribute.name
        : t(`AUTOMATION.ATTRIBUTES.${attribute.name}`),
    };
  });
};

const filterTypes = computed(() => {
  const event = eventName.value;
  if (!event || !props.automationTypes[event]) return [];

  const attributes = getTranslatedAttributes(props.automationTypes, event);

  return attributes.map(attr => {
    if (attr.disabled) {
      return { value: attr.key, label: attr.name, disabled: true };
    }

    const mappedInputType = INPUT_TYPE_MAP[attr.inputType] || 'plainText';
    const options = props.getConditionDropdownValues(attr.key) || [];

    const filterOperators = (attr.filterOperators || []).map(op => {
      const enriched = operators.value[op.value];
      if (enriched) return enriched;
      return {
        value: op.value,
        label: t(`FILTER.OPERATOR_LABELS.${op.value}`),
        hasInput: true,
        inputOverride: null,
        icon: h('span', { class: 'i-ph-equals-bold !text-n-blue-11' }),
      };
    });

    return {
      attributeKey: attr.key,
      value: attr.key,
      attributeName: attr.name,
      label: attr.name,
      icon: getAttributeIcon({
        attributeKey: attr.key,
        attributeDisplayType: attr.attributeDisplayType,
      }),
      inputType: mappedInputType,
      options,
      filterOperators,
      dataType: 'text',
      attributeModel: attr.customAttributeType || 'standard',
    };
  });
});

const automationRuleEvents = computed(() =>
  AUTOMATION_RULE_EVENTS.map(event => ({
    ...event,
    value: t(`AUTOMATION.EVENTS.${event.value}`),
  }))
);

const hasAutomationMutated = computed(() => {
  return Boolean(
    automation.value?.conditions[0]?.values ||
      automation.value?.actions[0]?.action_params?.length
  );
});

const automationActionTypes = computed(() => {
  const actionTypes = isCloudFeatureEnabled('sla')
    ? AUTOMATION_ACTION_TYPES
    : AUTOMATION_ACTION_TYPES.filter(({ key }) => key !== 'add_sla');

  return actionTypes.map(action => ({
    ...action,
    label: t(`AUTOMATION.ACTIONS.${action.label}`),
    icon: getActionIcon(action.key),
  }));
});

watch(
  () => automation.value,
  () => {
    if (Object.keys(errors.value).length) {
      errors.value = {};
    }
  },
  { deep: true }
);

const isConditionsValid = () => instantTriggerRef.value?.validate() ?? true;

const resetValidation = () => {
  errors.value = {};
  instantTriggerRef.value?.resetValidation();
};

const syncCustomAttributeTypes = () => {
  automation.value.conditions.forEach(condition => {
    const filterType = filterTypes.value.find(
      ft => ft.attributeKey === condition.attribute_key
    );
    condition.custom_attribute_type =
      filterType?.attributeModel === 'standard'
        ? ''
        : filterType?.attributeModel || '';
  });
};

const open = (executionDelay = null) => {
  resetValidation();
  syncDelayState(executionDelay);
  panelRef.value?.open();
};

const close = () => {
  resetValidation();
  panelRef.value?.close();
};

const emitSaveAutomation = () => {
  syncCustomAttributeTypes();
  const conditionsValid = isConditionsValid();
  errors.value = validateAutomation(automation.value);
  if (allowsDelayedExecution.value && executionDelayInvalid.value) {
    errors.value.execution_delay = true;
  }
  if (Object.keys(errors.value).length === 0 && conditionsValid) {
    const payload = generateAutomationPayload(automation.value);
    // The API rejects the param when the feature is off; existing values are kept server-side.
    if (!allowsDelayedExecution.value) delete payload.execution_delay;
    emit('save', payload, props.mode);
  }
};

defineExpose({ open, close });
</script>

<template>
  <SidePanel ref="panelRef" width="3xl" :title="$t(titleKey)">
    <div v-if="automation" class="flex flex-col w-full gap-6">
      <div class="flex flex-col">
        <woot-input
          v-model="automation.name"
          :label="$t('AUTOMATION.ADD.FORM.NAME.LABEL')"
          type="text"
          :class="{ error: errors.name }"
          :error="errors.name ? $t('AUTOMATION.ADD.FORM.NAME.ERROR') : ''"
          :placeholder="$t('AUTOMATION.ADD.FORM.NAME.PLACEHOLDER')"
        />
        <woot-input
          v-model="automation.description"
          :label="$t('AUTOMATION.ADD.FORM.DESC.LABEL')"
          type="text"
          :class="{ error: errors.description }"
          :error="
            errors.description ? $t('AUTOMATION.ADD.FORM.DESC.ERROR') : ''
          "
          :placeholder="$t('AUTOMATION.ADD.FORM.DESC.PLACEHOLDER')"
        />
      </div>
      <AutomationRunTypeSelector
        v-if="allowsDelayedExecution"
        v-model="isDelayed"
      />
      <AutomationWaitCondition
        v-if="isDelayed"
        :key="waitSectionKey"
        v-model:event-name="automation.event_name"
        v-model:conditions="automation.conditions"
        v-model:delay="delayMinutes"
        v-model:unit="delayUnit"
        :status-options="statusOptions"
        :inbox-options="inboxOptions"
        :is-saved-wait="isSavedWait"
        :has-error="Boolean(errors.execution_delay)"
      />
      <AutomationInstantTrigger
        v-else
        ref="instantTriggerRef"
        v-model:event-name="automation.event_name"
        v-model:conditions="automation.conditions"
        :events="automationRuleEvents"
        :filter-types="filterTypes"
        :errors="errors"
        :show-reset-message="!isEditMode && hasAutomationMutated"
        :append-new-condition="appendNewCondition"
        :remove-filter="removeFilter"
        :on-event-change="onEventChange"
      />
      <!-- Schedule section (time_elapsed event only) -->
      <section v-if="isTimerEvent">
        <label class="block mb-2 font-medium text-sm">
          {{ $t('AUTOMATION.SCHEDULE.LABEL') }}
        </label>
        <div
          class="grid gap-4 p-3 outline outline-1 rounded-xl -outline-offset-1 outline-n-weak dark:outline-n-strong"
        >
          <!-- Duration row -->
          <div class="flex items-center gap-3">
            <span class="text-sm text-n-slate-11 shrink-0">
              {{ $t('AUTOMATION.SCHEDULE.AFTER') }}
            </span>
            <input
              v-model.number="scheduleDuration"
              type="number"
              min="1"
              class="w-20 rounded-lg border border-n-weak px-2 py-1 text-sm"
            />
            <select
              v-model="scheduleUnit"
              class="m-0 rounded-lg border border-n-weak px-2 py-1 text-sm"
            >
              <option
                v-for="unit in scheduleDurationUnits"
                :key="unit.key"
                :value="unit.key"
              >
                {{ unit.label }}
              </option>
            </select>
          </div>
          <!-- Anchor row -->
          <div class="flex items-center gap-3">
            <span class="text-sm text-n-slate-11 shrink-0">
              {{ $t('AUTOMATION.SCHEDULE.SINCE') }}
            </span>
            <select
              v-model="automation.schedule_anchor"
              class="m-0 rounded-lg border border-n-weak px-2 py-1 text-sm"
            >
              <option
                v-for="opt in scheduleAnchorOptions"
                :key="opt.key"
                :value="opt.key"
              >
                {{ opt.label }}
              </option>
            </select>
          </div>
        </div>
      </section>
      <!-- End Schedule section -->
      <AutomationActions
        v-model="automation.actions"
        :action-types="automationActionTypes"
        :get-action-dropdown-values="getActionDropdownValues"
        :files="automation.files"
        :show-file-name="isEditMode"
        :errors="errors"
        :append-new-action="appendNewAction"
        :remove-action="removeAction"
        :reset-action="resetAction"
      />
    </div>
    <template #footer>
      <div class="flex flex-row justify-end w-full gap-2">
        <NextButton
          faded
          slate
          type="button"
          :label="$t(cancelKey)"
          @click="close"
        />
        <NextButton
          solid
          blue
          type="button"
          :label="$t(submitKey)"
          @click="emitSaveAutomation"
        />
      </div>
    </template>
  </SidePanel>
</template>
