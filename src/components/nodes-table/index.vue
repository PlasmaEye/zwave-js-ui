<template>
	<v-data-table
		v-if="managedNodes"
		v-model="managedNodes.selected"
		:headers="managedNodes.tableHeaders"
		:items="managedNodes.filteredItems"
		:footer-props="{
			itemsPerPageOptions: [10, 20, 50, 100, -1],
		}"
		:expanded.sync="expanded"
		:value="managedNodes.selected"
		:options="managedNodes.tableOptions"
		:custom-sort="sort"
		@update:options="managedNodes.tableOptions = $event"
		@input="managedNodes.selected = $event"
		@click:row="toggleExpanded($event)"
		item-key="id"
		class="elevation-1"
		show-expand
		show-select
	>
		<template v-slot:top>
			<v-row class="ma-2" justify-start>
				<v-col cols="12">
					<v-menu
						v-model="headersMenu"
						:close-on-content-click="false"
						@input="
							managedNodes.tableColumns =
								managedNodes.tableColumns
						"
					>
						<template v-slot:activator="{ on }">
							<v-btn color="primary" outlined v-on="on">
								<v-icon left small>table_chart</v-icon>
								Columns
							</v-btn>
						</template>
						<v-card>
							<v-card-text>
								<draggable
									v-model="managedNodes.tableColumns"
									handle=".handle"
								>
									<v-checkbox
										v-for="col in managedNodes.tableColumns"
										:key="col.name"
										v-model="col.visible"
										:value="col.visible"
										hide-details
										:label="
											managedNodes.propDefs[col.name]
												.label
										"
										:input-value="col.visible"
										@change="col.visible = !!$event"
									>
										<template v-slot:prepend>
											<v-icon
												class="handle"
												style="cursor: move"
												>drag_indicator</v-icon
											>
										</template>
									</v-checkbox>
								</draggable>
							</v-card-text>
							<v-card-actions>
								<v-btn
									@click.native="
										managedNodes.tableColumns =
											managedNodes.initialTableColumns
									"
									>Reset</v-btn
								>
							</v-card-actions>
						</v-card>
					</v-menu>
					<v-tooltip bottom>
						<template v-slot:activator="{ on }">
							<v-btn
								color="blue darken-1"
								text
								v-on="on"
								@click.native="
									managedNodes.setFilterToSelected()
								"
								:disabled="managedNodes.selected.length === 0"
								>Filter Selected</v-btn
							>
						</template>
						<span>Show only selected nodes</span>
					</v-tooltip>
					<v-tooltip bottom>
						<template v-slot:activator="{ on }">
							<v-btn
								color="blue darken-1"
								text
								v-on="on"
								@click.native="managedNodes.reset()"
								>Reset Table</v-btn
							>
						</template>
						<span>Reset all table settings</span>
					</v-tooltip>
				</v-col>
			</v-row>
		</template>
		<template
			v-for="column in managedNodes.tableHeaders"
			v-slot:[`header.${column.value}`]="{ header }"
		>
			<span :key="column.value">
				<column-filter
					:column="column"
					:value="managedNodes.filters[column.value]"
					:items="managedNodes.propValues[column.value]"
					:group-by="managedNodes.groupBy === [column.value]"
					@change="managedNodes.setPropFilter(column.value, $event)"
					@update:group-by="managedNodes.groupBy = $event"
				></column-filter>
				{{ header.text }}
			</span>
		</template>
		<template
			v-slot:[`group.header`]="{ group, headers, toggle, remove, isOpen }"
		>
			<td :colspan="headers.length">
				<v-btn @click="toggle" x-small icon :ref="group">
					<v-icon>{{ isOpen ? 'remove' : 'add' }}</v-icon>
				</v-btn>
				<span>{{ groupValue(group) }}</span>
				<v-btn x-small icon @click="remove"
					><v-icon>close</v-icon></v-btn
				>
			</td>
		</template>
		<template v-slot:[`item.id`]="{ item }">
			<div class="d-flex">
				<v-chip>{{ item.id.toString().padStart(3, '0') }}</v-chip>

				<reinterview-badge
					:node="item"
					v-on="$listeners"
				></reinterview-badge>
			</div>
		</template>
		<template v-slot:[`item.minBatteryLevel`]="{ item }">
			<rich-value :value="richValue(item, 'minBatteryLevel')" />
		</template>
		<template v-slot:[`item.manufacturer`]="{ item }">
			{{ item.manufacturer }}
		</template>
		<template v-slot:[`item.productDescription`]="{ item }">
			{{ item.productDescription }}
		</template>
		<template v-slot:[`item.productLabel`]="{ item }">
			{{ item.productLabel }}
		</template>
		<template v-slot:[`item.name`]="{ item }">
			{{ item.name || '' }}
		</template>
		<template v-slot:[`item.loc`]="{ item }">
			{{ item.loc || '' }}
		</template>
		<template v-slot:[`item.security`]="{ item }">
			<rich-value
				v-if="!item.isControllerNode"
				:value="richValue(item, 'security')"
			/>
			<div v-else></div>
		</template>
		<template v-slot:[`item.supportsBeaming`]="{ item }">
			<rich-value
				v-if="!item.isControllerNode"
				:value="richValue(item, 'supportsBeaming')"
			/>
			<div v-else></div>
		</template>
		<template v-slot:[`item.zwavePlusVersion`]="{ item }">
			<rich-value
				v-if="!item.isControllerNode"
				:value="richValue(item, 'zwavePlusVersion')"
			/>
			<div v-else></div>
		</template>
		<template v-slot:[`item.failed`]="{ item }">
			<rich-value
				v-if="!item.isControllerNode"
				:value="richValue(item, 'failed')"
			/>
			<div v-else></div>
		</template>
		<template v-slot:[`item.status`]="{ item }">
			<rich-value
				v-if="!item.isControllerNode"
				:value="richValue(item, 'status')"
			/>
			<div v-else></div>
		</template>
		<template v-slot:[`item.healProgress`]="{ item }">
			<div v-if="!item.isControllerNode">
				<v-progress-circular
					class="ml-3"
					v-if="item.healProgress === 'pending'"
					indeterminate
					size="20"
					color="primary"
				></v-progress-circular>

				<v-tooltip
					v-else-if="getHealIcon(item.healProgress) !== undefined"
					bottom
				>
					<template v-slot:activator="{ on }">
						<v-icon
							v-on="on"
							class="ml-3"
							v-text="getHealIcon(item.healProgress).icon"
							:color="getHealIcon(item.healProgress).color"
						></v-icon>
					</template>
					<span>{{ item.healProgress.toUpperCase() }}</span>
				</v-tooltip>
				<div v-else>{{ item.healProgress }}</div>
			</div>
			<div v-else></div>
		</template>
		<template v-slot:[`item.interviewStage`]="{ item }">
			<div v-if="!item.isControllerNode" style="text-align: center">
				<v-chip
					small
					:color="interviewStageColor(`${item.interviewStage}`)"
					text-color="white"
					>{{ item.interviewStage }}</v-chip
				>
				<v-progress-circular
					class="ml-3"
					v-if="item.interviewStage !== 'Complete'"
					indeterminate
					size="20"
					color="primary"
				></v-progress-circular>
			</div>
			<div v-else></div>
		</template>
		<template v-slot:[`item.firmwareVersion`]="{ item }">
			<div style="text-align: center">
				<div v-if="item.firmwareUpdate && !item.isControllerNode">
					<v-progress-circular
						:value="item.firmwareUpdate.progress"
						size="50"
						class="mt-1"
						color="primary"
					>
						<span class="caption">{{
							item.firmwareUpdate.progress
						}}</span>
					</v-progress-circular>
					<p class="caption font-weight-bold mb-0 mt-1">
						{{ item.firmwareUpdate.currentFile }}/{{
							item.firmwareUpdate.totalFiles
						}}: {{ getProgress(item) }}%
					</p>
				</div>
				<div
					style="white-space: pre"
					v-text="
						`FW: v${item.firmwareVersion}${
							item.sdkVersion ? `\nSDK: v${item.sdkVersion}` : ''
						}`
					"
					v-else
				></div>
			</div>
		</template>
		<template v-slot:[`item.lastActive`]="{ item }">
			<statistics-arrows :node="item"></statistics-arrows>
		</template>
		<template v-slot:[`expanded-item`]="{ headers, item, isMobile }">
			<td :colspan="isMobile ? 1 : headers.length">
				<expanded-node
					:headers="headers"
					:isMobile="isMobile"
					:node="item"
					:socket="socket"
					v-on="$listeners"
				/>
			</td>
		</template>
	</v-data-table>
</template>
<script src="./nodes-table.js"></script>
<style scoped src="./nodes-table.css"></style>
