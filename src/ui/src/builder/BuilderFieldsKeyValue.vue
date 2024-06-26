<template>
	<div
		ref="rootEl"
		class="BuilderFieldsOptions"
		tabindex="-1"
		:data-automation-key="props.fieldKey"
	>
		<div class="chipStackContainer">
			<div class="chipStack">
				<button
					class="chip"
					:class="{ active: mode == 'assisted' }"
					tabindex="0"
					@click="setMode('assisted')"
				>
					Static List
				</button>
				<button
					class="chip"
					tabindex="0"
					:class="{ active: mode == 'freehand' }"
					@click="setMode('freehand')"
				>
					JSON
				</button>
			</div>
		</div>

		<template v-if="mode == 'assisted'">
			<div class="staticList">
				<div
					v-for="(entryValue, entryKey) in assistedEntries"
					:key="entryKey"
					class="entry"
				>
					<div>{{ entryKey }} &middot; {{ entryValue }}</div>
					<button
						variant="subtle"
						@click="removeAssistedEntry(entryKey)"
					>
						<i class="material-symbols-outlined">delete</i>
					</button>
				</div>
			</div>
			<div class="formAdd">
				<BuilderTemplateInput
					ref="assistedKeyEl"
					class="inputKey"
					placeholder="Type a key..."
					:value="formAdd.key"
					@update:value="($event) => (formAdd.key = $event)"
					@keydown.enter="addAssistedEntry"
				/>
				<BuilderTemplateInput
					class="inputValue"
					placeholder="Type a value..."
					:value="formAdd.value"
					@update:value="($event) => (formAdd.value = $event)"
					@keydown.enter="addAssistedEntry"
				/>
				<button @click="addAssistedEntry">
					<i class="material-symbols-outlined">add</i>Add
				</button>
			</div>
		</template>

		<template v-if="mode == 'freehand'">
			<BuilderFieldsObject
				:component-id="component.id"
				:field-key="fieldKey"
			></BuilderFieldsObject>
		</template>
	</div>
</template>

<script setup lang="ts">
import {
	nextTick,
	onMounted,
	Ref,
	ref,
	toRefs,
	inject,
	computed,
	watch,
} from "vue";
import { Component } from "../writerTypes";
import BuilderFieldsObject from "./BuilderFieldsObject.vue";
import { useComponentActions } from "./useComponentActions";
import injectionKeys from "../injectionKeys";
import BuilderTemplateInput from "./BuilderTemplateInput.vue";

const wf = inject(injectionKeys.core);
const ssbm = inject(injectionKeys.builderManager);
const { setContentValue } = useComponentActions(wf, ssbm);

const props = defineProps<{
	componentId: Component["id"];
	fieldKey: string;
}>();
const { componentId, fieldKey } = toRefs(props);
const component = computed(() => wf.getComponentById(componentId.value));

const rootEl: Ref<HTMLElement> = ref(null);
const assistedKeyEl: Ref<HTMLInputElement> = ref(null);
type Mode = "assisted" | "freehand";
const mode: Ref<Mode> = ref(null);
const assistedEntries: Ref<Record<string, string>> = ref({});
const formAdd: Ref<{ key: string; value: string }> = ref({
	key: "",
	value: "",
});

const setMode = async (newMode: Mode) => {
	if (mode.value == newMode) return;
	mode.value = newMode;

	if (mode.value == "assisted") {
		handleSwitchToAssisted();
	}

	await nextTick();
};

const handleSwitchToAssisted = () => {
	let currentValue = component.value.content[fieldKey.value];

	if (!currentValue) return;

	let parsedValue: any;

	// Attempt to populate assisted from existing JSON data

	try {
		parsedValue = JSON.parse(currentValue);
		if (typeof parsedValue != "object") {
			throw "Invalid structure.";
		}
		assistedEntries.value = parsedValue;
	} catch {
		component.value.content[fieldKey.value] = JSON.stringify(
			assistedEntries.value,
			null,
			2,
		);
	}
};

const addAssistedEntry = () => {
	const { key, value } = formAdd.value;
	if (key === "" || value === "") return;
	assistedEntries.value[key] = value;
	setContentValue(
		component.value.id,
		fieldKey.value,
		JSON.stringify(assistedEntries.value, null, 2),
	);
	formAdd.value = { key: "", value: "" };
	assistedKeyEl.value.focus();
};

const removeAssistedEntry = (key: string) => {
	delete assistedEntries.value[key];

	if (Object.keys(assistedEntries.value).length == 0) {
		setContentValue(component.value.id, fieldKey.value, undefined);
		return;
	}

	setContentValue(
		component.value.id,
		fieldKey.value,
		JSON.stringify(assistedEntries.value, null, 2),
	);
};

/**
 * Watcher for external mutations (like undo/redo).
 */
watch(
	() => component.value?.content[fieldKey.value],
	async (currentValue) => {
		if (!component.value) return;
		let parsedValue: any;

		try {
			parsedValue = JSON.parse(currentValue);
			assistedEntries.value = parsedValue;
		} catch {
			// If parsing fails, preserve the previous assistedEntries value
		}
	},
);

onMounted(async () => {
	const currentValue = component.value.content[fieldKey.value];
	let parsedValue: any;

	if (!currentValue) {
		setMode("assisted");
		return;
	}

	// Attempt assisted mode first, if the JSON can be parsed into an object

	try {
		parsedValue = JSON.parse(currentValue);
		if (typeof parsedValue != "object") {
			throw "Invalid structure.";
		}
		assistedEntries.value = parsedValue;
		setMode("assisted");
	} catch {
		// Fall back to freehand if assisted mode isn't possible
		setMode("freehand");
	}
});
</script>

<style scoped>
@import "./sharedStyles.css";

.chipStackContainer {
	padding: 12px;
	margin-top: 4px;
	padding-bottom: 12px;
	border-bottom: 1px solid var(--builderSeparatorColor);
}

.staticList {
	padding: 12px;
	border-bottom: 1px solid var(--builderSeparatorColor);
}

.staticList:empty::before {
	content: "No entries yet.";
}

.staticList .entry {
	display: flex;
	align-items: center;
	gap: 8px;
	width: 100%;
	max-width: 100%;
}

.staticList .entry div {
	overflow: hidden;
	padding-top: 4px;
	padding-bottom: 4px;
	flex: 1 1 auto;
}

.staticList .entry button {
	flex: 0 0 auto;
}

.formAdd {
	padding: 12px;
}

.BuilderTemplateInput {
	margin-bottom: 8px;
}
</style>
