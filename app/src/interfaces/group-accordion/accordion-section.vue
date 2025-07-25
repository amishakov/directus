<script setup lang="ts">
import { Field, ValidationError } from '@directus/types';
import { isNil, merge } from 'lodash';
import { computed } from 'vue';
import { useI18n } from 'vue-i18n';

const props = withDefaults(
	defineProps<{
		field: Field;
		fields: Field[];
		values: Record<string, unknown>;
		initialValues: Record<string, unknown>;
		disabled?: boolean;
		batchMode?: boolean;
		batchActiveFields?: string[];
		primaryKey: number | string;
		loading?: boolean;
		validationErrors?: ValidationError[];
		badge?: string;
		group: string;
		multiple?: boolean;
		direction?: string;
	}>(),
	{
		batchActiveFields: () => [],
		validationErrors: () => [],
	},
);

const emit = defineEmits<{
	(e: 'apply', value: Record<string, unknown>): void;
	(e: 'toggleAll'): void;
}>();

const { t } = useI18n();

const fieldsInSection = computed(() => {
	const fields: Field[] = [merge({}, props.field, { hideLabel: true })];

	if (props.field.meta?.special?.includes('group')) {
		fields.push(...getFieldsForGroup(props.field.meta?.field));
	}

	return fields;
});

const edited = computed(() => {
	if (!props.values) return false;

	const editedFields = Object.keys(props.values);
	return fieldsInSection.value.some((field) => editedFields.includes(field.field));
});

const validationMessage = computed(() => {
	const validationError = props.validationErrors.find((error) => error.field === props.field.field);
	if (validationError === undefined) return;

	if (validationError.code === 'RECORD_NOT_UNIQUE') {
		return t('validationError.unique');
	} else {
		return t(`validationError.${validationError.type}`, validationError);
	}
});

function handleModifier(event: MouseEvent, toggle: () => void) {
	if (props.multiple === false) {
		toggle();
		return;
	}

	if (event.shiftKey) {
		emit('toggleAll');
	} else {
		toggle();
	}
}

function getFieldsForGroup(group: null | string, passed: string[] = []): Field[] {
	const fieldsInGroup: Field[] = props.fields.filter((field) => {
		return field.meta?.group === group || (group === null && isNil(field.meta));
	});

	for (const field of fieldsInGroup) {
		if (field.meta?.special?.includes('group') && !passed.includes(field.meta!.field)) {
			passed.push(field.meta!.field);
			fieldsInGroup.push(...getFieldsForGroup(field.meta!.field, passed));
		}
	}

	return fieldsInGroup;
}
</script>

<template>
	<v-item v-if="!field.meta?.hidden" :value="field.field" scope="group-accordion" class="accordion-section">
		<template #default="{ active, toggle }">
			<button
				type="button"
				class="label type-title"
				:class="{ active, edited }"
				@click="handleModifier($event, toggle)"
			>
				<span v-if="edited" v-tooltip="t('edited')" class="edit-dot"></span>
				<v-icon class="icon" :class="{ active }" name="expand_more" />
				<span class="field-name">{{ field.name }}</span>
				<v-icon v-if="field.meta?.required === true" class="required" sup name="star" filled />
				<v-chip v-if="badge" x-small>{{ badge }}</v-chip>
				<v-icon v-if="!active && validationMessage" v-tooltip="validationMessage" class="warning" name="error" small />
			</button>

			<transition-expand>
				<div v-if="active" class="fields">
					<v-form
						:initial-values="initialValues"
						:fields="fieldsInSection"
						:model-value="values"
						:primary-key="primaryKey"
						:group="group"
						:validation-errors="validationErrors"
						:loading="loading"
						:batch-mode="batchMode"
						:disabled="disabled"
						:direction="direction"
						:show-no-visible-fields="false"
						:show-validation-errors="false"
						@update:model-value="$emit('apply', $event)"
					/>
				</div>
			</transition-expand>
		</template>
	</v-item>
</template>

<style lang="scss" scoped>
.accordion-section {
	border-block-start: var(--theme--border-width) solid var(--theme--form--field--input--border-color);

	&:last-child {
		border-block-end: var(--theme--border-width) solid var(--theme--form--field--input--border-color);
	}
}

.label {
	position: relative;
	display: flex;
	align-items: center;
	inline-size: 100%;
	margin: 8px 0;

	cursor: pointer;

	&:hover,
	&.active {
		.field-name,
		.icon {
			color: var(--theme--form--field--input--foreground);
		}
	}

	.field-name,
	.icon {
		color: var(--theme--form--field--input--foreground-subdued);
		transition: color var(--fast) var(--transition);
	}

	.required {
		--v-icon-color: var(--theme--primary);

		margin-block-start: -12px;
		margin-inline-start: 2px;
	}

	.v-chip {
		margin: 0;
		margin-inline-start: 8px;
	}

	.edit-dot {
		position: absolute;
		inset-block-start: 14px;
		inset-inline-start: -7px;
		display: block;
		inline-size: 4px;
		block-size: 4px;
		background-color: var(--theme--form--field--input--foreground-subdued);
		border-radius: 4px;
		content: '';
	}
}

.icon {
	margin-inline-end: 12px;
	transform: rotate(-90deg);
	transition: transform var(--fast) var(--transition);

	&.active {
		transform: rotate(0);
	}
}

.warning {
	margin-inline-start: 8px;
	color: var(--theme--danger);
}

.fields {
	margin: var(--theme--form--row-gap) 0;
}
</style>
