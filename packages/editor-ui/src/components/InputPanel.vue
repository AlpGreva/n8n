<template>
	<RunData
		:nodeUi="currentNode"
		:runIndex="runIndex"
		:linkedRuns="linkedRuns"
		:canLinkRuns="!mappedNode && canLinkRuns"
		:tooMuchDataTitle="$locale.baseText('ndv.input.tooMuchData.title')"
		:noDataInBranchMessage="$locale.baseText('ndv.input.noOutputDataInBranch')"
		:isExecuting="isExecutingPrevious"
		:executingMessage="$locale.baseText('ndv.input.executingPrevious')"
		:sessionId="sessionId"
		:overrideOutputs="connectedCurrentNodeOutputs"
		:mappingEnabled="isMappingEnabled"
		:distanceFromActive="currentNodeDepth"
		:isProductionExecutionPreview="isProductionExecutionPreview"
		paneType="input"
		@itemHover="$emit('itemHover', $event)"
		@linkRun="onLinkRun"
		@unlinkRun="onUnlinkRun"
		@runChange="onRunIndexChange"
		@tableMounted="$emit('tableMounted', $event)"
		data-test-id="ndv-input-panel"
	>
		<template #header>
			<div :class="$style.titleSection">
				<n8n-select
					v-if="parentNodes.length"
					teleported
					size="small"
					:modelValue="currentNodeName"
					@update:modelValue="onInputNodeChange"
					:no-data-text="$locale.baseText('ndv.input.noNodesFound')"
					:placeholder="$locale.baseText('ndv.input.parentNodes')"
					filterable
					data-test-id="ndv-input-select"
				>
					<template #prepend>
						<span :class="$style.title">{{ $locale.baseText('ndv.input') }}</span>
					</template>
					<n8n-option
						v-for="node of parentNodes"
						:value="node.name"
						:key="node.name"
						class="node-option"
						:label="`${truncate(node.name)} ${getMultipleNodesText(node.name)}`"
						data-test-id="ndv-input-option"
					>
						<span>{{ truncate(node.name) }}&nbsp;</span>
						<span v-if="getMultipleNodesText(node.name)">{{
							getMultipleNodesText(node.name)
						}}</span>
						<span v-else>{{
							$locale.baseText('ndv.input.nodeDistance', { adjustToNumber: node.depth })
						}}</span>
					</n8n-option>
				</n8n-select>
				<span v-else :class="$style.title">{{ $locale.baseText('ndv.input') }}</span>
				<n8n-radio-buttons
					v-if="isActiveNodeConfig && !readOnly"
					:options="inputModes"
					:modelValue="inputMode"
					@update:modelValue="onInputModeChange"
				/>
			</div>
		</template>
		<template #before-data v-if="isMappingMode">
			<!--
						Hide the run linking buttons for both input and ouput panels when in 'Mapping Mode' because the run indices wouldn't match.
						Although this is not the most elegant solution, it's straightforward and simpler than introducing a new props and logic to handle this.
				-->
			<component :is="'style'">button.linkRun { display: none }</component>
			<div :class="$style.mappedNode">
				<n8n-select
					:modelValue="mappedNode"
					@update:modelValue="onMappedNodeSelected"
					size="small"
					@click.stop
					teleported
				>
					<template #prepend>{{ $locale.baseText('ndv.input.previousNode') }}</template>
					<n8n-option
						v-for="nodeName in rootNodesParents"
						:key="nodeName"
						:label="nodeName"
						:value="nodeName"
					/>
				</n8n-select>
			</div>
		</template>
		<template #node-not-run>
			<div
				:class="$style.noOutputData"
				v-if="(isActiveNodeConfig && rootNode) || parentNodes.length"
			>
				<n8n-text tag="div" :bold="true" color="text-dark" size="large">{{
					$locale.baseText('ndv.input.noOutputData.title')
				}}</n8n-text>
				<n8n-tooltip v-if="!readOnly" :visible="showDraggableHint && showDraggableHintWithDelay">
					<template #content>
						<div
							v-html="
								$locale.baseText('dataMapping.dragFromPreviousHint', {
									interpolate: { name: focusedMappableInput },
								})
							"
						></div>
					</template>
					<NodeExecuteButton
						type="secondary"
						:transparent="true"
						:nodeName="isActiveNodeConfig ? rootNode : currentNodeName"
						:label="$locale.baseText('ndv.input.noOutputData.executePrevious')"
						@execute="onNodeExecute"
						telemetrySource="inputs"
						data-test-id="execute-previous-node"
					/>
				</n8n-tooltip>
				<n8n-text v-if="!readOnly" tag="div" size="small">
					{{ $locale.baseText('ndv.input.noOutputData.hint') }}
				</n8n-text>
			</div>
			<div :class="$style.notConnected" v-else>
				<div>
					<WireMeUp />
				</div>
				<n8n-text tag="div" :bold="true" color="text-dark" size="large">{{
					$locale.baseText('ndv.input.notConnected.title')
				}}</n8n-text>
				<n8n-text tag="div">
					{{ $locale.baseText('ndv.input.notConnected.message') }}
					<a
						href="https://docs.n8n.io/workflows/connections/"
						target="_blank"
						@click="onConnectionHelpClick"
					>
						{{ $locale.baseText('ndv.input.notConnected.learnMore') }}
					</a>
				</n8n-text>
			</div>
		</template>

		<template #no-output-data>
			<n8n-text tag="div" :bold="true" color="text-dark" size="large">{{
				$locale.baseText('ndv.input.noOutputData')
			}}</n8n-text>
		</template>

		<template #recovered-artificial-output-data>
			<div :class="$style.recoveredOutputData">
				<n8n-text tag="div" :bold="true" color="text-dark" size="large">{{
					$locale.baseText('executionDetails.executionFailed.recoveredNodeTitle')
				}}</n8n-text>
				<n8n-text>
					{{ $locale.baseText('executionDetails.executionFailed.recoveredNodeMessage') }}
				</n8n-text>
			</div>
		</template>
	</RunData>
</template>

<script lang="ts">
import { defineComponent } from 'vue';
import { mapStores } from 'pinia';
import type { INodeUi } from '@/Interface';
import { NodeHelpers, NodeConnectionType } from 'n8n-workflow';
import type { ConnectionTypes, IConnectedNode, INodeTypeDescription, Workflow } from 'n8n-workflow';
import RunData from './RunData.vue';
import { workflowHelpers } from '@/mixins/workflowHelpers';
import NodeExecuteButton from './NodeExecuteButton.vue';
import WireMeUp from './WireMeUp.vue';
import {
	CRON_NODE_TYPE,
	INTERVAL_NODE_TYPE,
	MANUAL_TRIGGER_NODE_TYPE,
	START_NODE_TYPE,
} from '@/constants';
import { useWorkflowsStore } from '@/stores/workflows.store';
import { useNDVStore } from '@/stores/ndv.store';
import { useNodeTypesStore } from '@/stores/nodeTypes.store';

type MappingMode = 'debugging' | 'mapping';

export default defineComponent({
	name: 'InputPanel',
	mixins: [workflowHelpers],
	components: { RunData, NodeExecuteButton, WireMeUp },
	props: {
		currentNodeName: {
			type: String,
		},
		runIndex: {
			type: Number,
		},
		linkedRuns: {
			type: Boolean,
		},
		workflow: {},
		canLinkRuns: {
			type: Boolean,
		},
		sessionId: {
			type: String,
		},
		readOnly: {
			type: Boolean,
		},
		isProductionExecutionPreview: {
			type: Boolean,
			default: false,
		},
	},
	data() {
		return {
			showDraggableHintWithDelay: false,
			draggableHintShown: false,
			inputMode: 'debugging' as MappingMode,
			mappedNode: null as string | null,
			inputModes: [
				{ value: 'mapping', label: this.$locale.baseText('ndv.input.mapping') },
				{ value: 'debugging', label: this.$locale.baseText('ndv.input.debugging') },
			],
		};
	},
	computed: {
		...mapStores(useNodeTypesStore, useNDVStore, useWorkflowsStore),
		focusedMappableInput(): string {
			return this.ndvStore.focusedMappableInput;
		},
		isUserOnboarded(): boolean {
			return this.ndvStore.isMappingOnboarded;
		},
		isMappingMode(): boolean {
			return this.isActiveNodeConfig && this.inputMode === 'mapping';
		},
		showDraggableHint(): boolean {
			const toIgnore = [
				START_NODE_TYPE,
				MANUAL_TRIGGER_NODE_TYPE,
				CRON_NODE_TYPE,
				INTERVAL_NODE_TYPE,
			];
			if (!this.currentNode || toIgnore.includes(this.currentNode.type)) {
				return false;
			}

			return !!this.focusedMappableInput && !this.isUserOnboarded;
		},
		isActiveNodeConfig(): boolean {
			let inputs = this.activeNodeType?.inputs ?? [];
			let outputs = this.activeNodeType?.outputs ?? [];
			if (this.activeNode !== null && this.currentWorkflow !== null) {
				const node = this.currentWorkflow.getNode(this.activeNode.name);
				inputs = NodeHelpers.getNodeInputs(this.currentWorkflow, node!, this.activeNodeType!);
				outputs = NodeHelpers.getNodeOutputs(this.currentWorkflow, node!, this.activeNodeType!);
			} else {
				// If we can not figure out the node type we set no outputs
				if (!Array.isArray(inputs)) {
					inputs = [] as ConnectionTypes[];
				}
				if (!Array.isArray(outputs)) {
					outputs = [] as ConnectionTypes[];
				}
			}

			if (
				(inputs.length === 0 ||
					inputs.find((inputName) => inputName !== NodeConnectionType.Main)) &&
				outputs.find((outputName) => outputName !== NodeConnectionType.Main)
			) {
				return true;
			}

			return false;
		},
		isMappingEnabled(): boolean {
			if (this.readOnly) return false;

			// Mapping is only enabled in mapping mode for config nodes and if node to map is selected
			if (this.isActiveNodeConfig) return this.isMappingMode && this.mappedNode !== null;

			return true;
		},
		isExecutingPrevious(): boolean {
			if (!this.workflowRunning) {
				return false;
			}
			const triggeredNode = this.workflowsStore.executedNode;
			const executingNode = this.workflowsStore.executingNode;

			if (
				this.activeNode &&
				triggeredNode === this.activeNode.name &&
				!this.workflowsStore.isNodeExecuting(this.activeNode.name)
			) {
				return true;
			}

			if (executingNode.length || triggeredNode) {
				return !!this.parentNodes.find(
					(node) => this.workflowsStore.isNodeExecuting(node.name) || node.name === triggeredNode,
				);
			}
			return false;
		},
		workflowRunning(): boolean {
			return this.uiStore.isActionActive('workflowRunning');
		},
		currentWorkflow(): Workflow {
			return this.workflow as Workflow;
		},
		activeNode(): INodeUi | null {
			return this.ndvStore.activeNode;
		},

		rootNode(): string {
			const workflow = this.currentWorkflow;
			const rootNodes = workflow.getChildNodes(this.activeNode.name, 'ALL_NON_MAIN');

			return rootNodes[0];
		},
		rootNodesParents(): string[] {
			const workflow = this.currentWorkflow;
			const parentNodes = [...workflow.getParentNodes(this.rootNode, 'main')].reverse();

			return parentNodes;
		},
		currentNode(): INodeUi | null {
			if (this.isActiveNodeConfig) {
				// if we're mapping node we want to show the output of the mapped node
				if (this.mappedNode) {
					return this.workflowsStore.getNodeByName(this.mappedNode);
				}

				// in debugging mode data does get set manually and is only for debugging
				// so we want to force the node to be the active node to make sure we show the correct data
				return this.activeNode;
			}

			return this.workflowsStore.getNodeByName(this.currentNodeName);
		},
		connectedCurrentNodeOutputs(): number[] | undefined {
			const search = this.parentNodes.find(({ name }) => name === this.currentNodeName);
			if (search) {
				return search.indicies;
			}
			return undefined;
		},
		parentNodes(): IConnectedNode[] {
			if (!this.activeNode) {
				return [];
			}
			const nodes: IConnectedNode[] = (this.workflow as Workflow).getParentNodesByDepth(
				this.activeNode.name,
			);

			return nodes.filter(
				({ name }, i) =>
					this.activeNode &&
					name !== this.activeNode.name &&
					nodes.findIndex((node) => node.name === name) === i,
			);
		},
		currentNodeDepth(): number {
			const node = this.parentNodes.find(
				(node) => this.currentNode && node.name === this.currentNode.name,
			);
			return node ? node.depth : -1;
		},
		activeNodeType(): INodeTypeDescription | null {
			if (!this.activeNode) return null;

			return this.nodeTypesStore.getNodeType(this.activeNode.type, this.activeNode.typeVersion);
		},
		isMultiInputNode(): boolean {
			return this.activeNodeType !== null && this.activeNodeType.inputs.length > 1;
		},
	},
	methods: {
		onInputModeChange(val: MappingMode) {
			this.inputMode = val;
		},
		onMappedNodeSelected(val: string) {
			this.mappedNode = val;

			this.onRunIndexChange(0);
			this.onUnlinkRun();
		},
		getMultipleNodesText(nodeName?: string): string {
			if (
				!nodeName ||
				!this.isMultiInputNode ||
				!this.activeNode ||
				this.activeNodeType?.inputNames === undefined
			)
				return '';

			const activeNodeConnections =
				this.currentWorkflow.connectionsByDestinationNode[this.activeNode.name].main || [];
			// Collect indexes of connected nodes
			const connectedInputIndexes = activeNodeConnections.reduce((acc: number[], node, index) => {
				if (node[0] && node[0].node === nodeName) return [...acc, index];
				return acc;
			}, []);

			// Match connected input indexes to their names specified by active node
			const connectedInputs = connectedInputIndexes.map(
				(inputIndex) => this.activeNodeType?.inputNames?.[inputIndex],
			);

			if (connectedInputs.length === 0) return '';

			return `(${connectedInputs.join(' & ')})`;
		},
		onNodeExecute() {
			this.$emit('execute');
			if (this.activeNode) {
				this.$telemetry.track('User clicked ndv button', {
					node_type: this.activeNode.type,
					workflow_id: this.workflowsStore.workflowId,
					session_id: this.sessionId,
					pane: 'input',
					type: 'executePrevious',
				});
			}
		},
		onRunIndexChange(run: number) {
			this.$emit('runChange', run);
		},
		onLinkRun() {
			this.$emit('linkRun');
		},
		onUnlinkRun() {
			this.$emit('unlinkRun');
		},
		onInputNodeChange(value: string) {
			const index = this.parentNodes.findIndex((node) => node.name === value) + 1;
			this.$emit('changeInputNode', value, index);
		},
		onConnectionHelpClick() {
			if (this.activeNode) {
				this.$telemetry.track('User clicked ndv link', {
					node_type: this.activeNode.type,
					workflow_id: this.workflowsStore.workflowId,
					session_id: this.sessionId,
					pane: 'input',
					type: 'not-connected-help',
				});
			}
		},
		truncate(nodeName: string) {
			const truncated = nodeName.substring(0, 30);
			if (truncated.length < nodeName.length) {
				return `${truncated}...`;
			}
			return truncated;
		},
	},
	watch: {
		inputMode: {
			handler(val) {
				this.onRunIndexChange(-1);
				if (val === 'mapping') {
					this.onUnlinkRun();
					this.mappedNode = this.rootNodesParents[0];
				}
			},
			immediate: true,
		},
		showDraggableHint(curr: boolean, prev: boolean) {
			if (curr && !prev) {
				setTimeout(() => {
					if (this.draggableHintShown) {
						return;
					}
					this.showDraggableHintWithDelay = this.showDraggableHint;
					if (this.showDraggableHintWithDelay) {
						this.draggableHintShown = true;

						this.$telemetry.track('User viewed data mapping tooltip', {
							type: 'unexecuted input pane',
						});
					}
				}, 1000);
			} else if (!curr) {
				this.showDraggableHintWithDelay = false;
			}
		},
	},
});
</script>

<style lang="scss" module>
.mappedNode {
	width: max-content;
	padding: 0 var(--spacing-s) var(--spacing-s);
}
.titleSection {
	display: flex;
	max-width: 300px;
	align-items: center;

	> * {
		margin-right: var(--spacing-2xs);
	}
}
.inputModeTab {
	margin-left: auto;
}
.noOutputData {
	max-width: 180px;

	> *:first-child {
		margin-bottom: var(--spacing-m);
	}

	> * {
		margin-bottom: var(--spacing-2xs);
	}
}

.recoveredOutputData {
	margin: auto;
	max-width: 250px;
	text-align: center;

	> *:first-child {
		margin-bottom: var(--spacing-m);
	}
}

.notConnected {
	max-width: 300px;

	> *:first-child {
		margin-bottom: var(--spacing-m);
	}

	> * {
		margin-bottom: var(--spacing-2xs);
	}
}

.title {
	text-transform: uppercase;
	color: var(--color-text-light);
	letter-spacing: 3px;
	font-size: var(--font-size-s);
	font-weight: var(--font-weight-bold);
}
</style>

<style lang="scss" scoped>
.node-option {
	font-weight: var(--font-weight-regular) !important;

	span {
		color: var(--color-text-light);
	}

	&.selected > span {
		color: var(--color-primary);
	}
}
</style>
