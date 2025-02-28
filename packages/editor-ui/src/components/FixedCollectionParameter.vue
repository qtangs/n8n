<template>
	<div @keydown.stop class="fixed-collection-parameter">
		<div v-if="getProperties.length === 0" class="no-items-exist">
			<n8n-text size="small">{{ $locale.baseText('fixedCollectionParameter.currentlyNoItemsExist') }}</n8n-text>
		</div>

		<div
			v-for="property in getProperties"
			:key="property.name"
			class="fixed-collection-parameter-property"
		>
			<n8n-input-label
				v-if="property.displayName !== '' && (parameter.options && parameter.options.length !== 1)"
				:label="$locale.nodeText().inputLabelDisplayName(property, path)"
				:underline="true"
				size="small"
				color="text-dark"
			/>
			<div v-if="multipleValues === true">
				<div
					v-for="(value, index) in values[property.name]"
					:key="property.name + index"
					class="parameter-item"
				>
					<div class="parameter-item-wrapper">
						<div class="delete-option" v-if="!isReadOnly">
							<font-awesome-icon
								icon="trash"
								class="reset-icon clickable"
								:title="$locale.baseText('fixedCollectionParameter.deleteItem')"
								@click="deleteOption(property.name, index)"
							/>
							<div v-if="sortable" class="sort-icon">
								<font-awesome-icon
									v-if="index !== 0"
									icon="angle-up"
									class="clickable"
									:title="$locale.baseText('fixedCollectionParameter.moveUp')"
									@click="moveOptionUp(property.name, index)"
								/>
								<font-awesome-icon
									v-if="index !== (values[property.name].length - 1)"
									icon="angle-down"
									class="clickable"
									:title="$locale.baseText('fixedCollectionParameter.moveDown')"
									@click="moveOptionDown(property.name, index)"
								/>
							</div>
						</div>
						<parameter-input-list
							:parameters="property.values"
							:nodeValues="nodeValues"
							:path="getPropertyPath(property.name, index)"
							:hideDelete="true"
							@valueChanged="valueChanged"
						/>
					</div>
				</div>
			</div>
			<div v-else class="parameter-item">
				<div class="parameter-item-wrapper">
					<div class="delete-option" v-if="!isReadOnly">
						<font-awesome-icon
							icon="trash"
							class="reset-icon clickable"
							:title="$locale.baseText('fixedCollectionParameter.deleteItem')"
							@click="deleteOption(property.name)"
						/>
					</div>
					<parameter-input-list
						:parameters="property.values"
						:nodeValues="nodeValues"
						:path="getPropertyPath(property.name)"
						class="parameter-item"
						@valueChanged="valueChanged"
						:hideDelete="true"
					/>
				</div>
			</div>
		</div>

		<div v-if="parameterOptions.length > 0 && !isReadOnly">
			<n8n-button
				v-if="parameter.options.length === 1"
				type="tertiary"
				block
				@click="optionSelected(parameter.options[0].name)"
				:label="getPlaceholderText"
			/>
			<div v-else class="add-option">
				<n8n-select
					v-model="selectedOption"
					:placeholder="getPlaceholderText"
					size="small"
					@change="optionSelected"
					filterable
				>
					<n8n-option
						v-for="item in parameterOptions"
						:key="item.name"
						:label="$locale.nodeText().collectionOptionDisplayName(parameter, item, path)"
						:value="item.name"
					></n8n-option>
				</n8n-select>
			</div>
		</div>
	</div>
</template>

<script lang="ts">
import {
	IUpdateInformation,
} from '@/Interface';

import {
	deepCopy,
	INodeParameters,
	INodePropertyCollection,
	NodeParameterValue,
} from 'n8n-workflow';

import { get } from 'lodash';

import { genericHelpers } from '@/components/mixins/genericHelpers';

import mixins from 'vue-typed-mixins';
import {Component} from "vue";

export default mixins(genericHelpers)
	.extend({
		name: 'FixedCollectionParameter',
		props: [
			'nodeValues', // INodeParameters
			'parameter', // INodeProperties
			'path', // string
			'values', // INodeParameters
		],
		components: {
			ParameterInputList: () => import('./ParameterInputList.vue') as Promise<Component>,
		},
		data() {
			return {
				selectedOption: undefined,
			};
		},

		computed: {
			getPlaceholderText(): string {
				const placeholder = this.$locale.nodeText().placeholder(this.parameter, this.path);
				return placeholder ? placeholder : this.$locale.baseText('fixedCollectionParameter.choose');
			},
			getProperties(): INodePropertyCollection[] {
				const returnProperties = [];
				let tempProperties;
				for (const name of this.propertyNames) {
					tempProperties = this.getOptionProperties(name);
					if (tempProperties !== undefined) {
						returnProperties.push(tempProperties);
					}
				}
				return returnProperties;
			},
			multipleValues(): boolean {
				if (this.parameter.typeOptions !== undefined && this.parameter.typeOptions.multipleValues === true) {
					return true;
				}
				return false;
			},

			parameterOptions(): INodePropertyCollection[] {
				if (this.multipleValues === true) {
					return this.parameter.options;
				}

				return (this.parameter.options as INodePropertyCollection[]).filter((option) => {
					return !this.propertyNames.includes(option.name);
				});
			},
			propertyNames(): string[] {
				if (this.values) {
					return Object.keys(this.values);
				}
				return [];
			},
			sortable(): string {
				return this.parameter.typeOptions && this.parameter.typeOptions.sortable;
			},
		},
		methods: {
			deleteOption(optionName: string, index?: number) {
				const currentOptionsOfSameType = this.values[optionName];
				if (!currentOptionsOfSameType || currentOptionsOfSameType.length > 1) {
					// it's not the only option of this type, so just remove it.
					this.$emit('valueChanged', {
						name: this.getPropertyPath(optionName, index),
						value: undefined,
					});
				} else {
					// it's the only option, so remove the whole type
					this.$emit('valueChanged', {
						name: this.getPropertyPath(optionName),
						value: undefined,
					});
				}
			},
			getPropertyPath(name: string, index?: number) {
				return `${this.path}.${name}` + (index !== undefined ? `[${index}]` : '');
			},
			getOptionProperties(optionName: string): INodePropertyCollection | undefined {
				for (const option of this.parameter.options) {
					if (option.name === optionName) {
						return option;
					}
				}

				return undefined;
			},
			moveOptionDown(optionName: string, index: number) {
				this.values[optionName].splice(index + 1, 0, this.values[optionName].splice(index, 1)[0]);

				const parameterData = {
					name: this.getPropertyPath(optionName),
					value: this.values[optionName],
				};

				this.$emit('valueChanged', parameterData);
			},
			moveOptionUp(optionName: string, index: number) {
				this.values[optionName].splice(index - 1, 0, this.values[optionName].splice(index, 1)[0]);

				const parameterData = {
					name: this.getPropertyPath(optionName),
					value: this.values[optionName],
				};

				this.$emit('valueChanged', parameterData);
			},
			optionSelected(optionName: string) {
				const option = this.getOptionProperties(optionName);
				if (option === undefined) {
					return;
				}
				const name = `${this.path}.${option.name}`;

				const newParameterValue: INodeParameters = {};

				for (const optionParameter of option.values) {
					if (optionParameter.type === 'fixedCollection' && optionParameter.typeOptions !== undefined && optionParameter.typeOptions.multipleValues === true) {
						newParameterValue[optionParameter.name] = {};
					} else if (optionParameter.typeOptions !== undefined && optionParameter.typeOptions.multipleValues === true) {
						// Multiple values are allowed so append option to array
						newParameterValue[optionParameter.name] = get(this.nodeValues, `${this.path}.${optionParameter.name}`, []);
						if (Array.isArray(optionParameter.default)) {
							(newParameterValue[optionParameter.name] as INodeParameters[]).push(...deepCopy(optionParameter.default as INodeParameters[]));
						} else if (optionParameter.default !== '' && typeof optionParameter.default !== 'object') {
							(newParameterValue[optionParameter.name] as NodeParameterValue[]).push(deepCopy(optionParameter.default));
						}
					} else {
						// Add a new option
						newParameterValue[optionParameter.name] = deepCopy(optionParameter.default);
					}
				}

				let newValue;
				if (this.multipleValues === true) {
					newValue = get(this.nodeValues, name, []);

					newValue.push(newParameterValue);
				} else {
					newValue = newParameterValue;
				}

				const parameterData = {
					name,
					value: newValue,
				};

				this.$emit('valueChanged', parameterData);
				this.selectedOption = undefined;
			},
			valueChanged(parameterData: IUpdateInformation) {
				this.$emit('valueChanged', parameterData);
			},
		},
	});
</script>

<style scoped lang="scss">
::v-deep {
	.button {
		--button-background-color: var(--color-background-base);
		--button-border-color: var(--color-foreground-base);
	}
}

.fixed-collection-parameter {
	padding-left: var(--spacing-s);
}

.fixed-collection-parameter-property {
	margin: var(--spacing-xs) 0;
}

.delete-option {
	display: none;
	position: absolute;
	z-index: 999;
	color: #f56c6c;
	left: 0;
	top: 0.5em;
	width: 15px;
	height: 100%;
}

.parameter-item:hover > .parameter-item-wrapper > .delete-option {
	display: block;
}

.parameter-item {
	position: relative;
	padding: 0 0 0 1em;

	+ .parameter-item {
		.parameter-item-wrapper {
			border-top: 1px dashed #999;

			.delete-option {
				top: 14px;
			}
		}
	}
}

.no-items-exist {
	margin: var(--spacing-xs) 0;
}

.sort-icon {
	display: flex;
	flex-direction: column;
	margin-left: 1px;
	margin-top: 0.5em;
}
</style>
