<script lang="ts">
	import { sendUserToast } from '$lib/utils'
	import { faFileExport, faFileImport, faGlobe } from '@fortawesome/free-solid-svg-icons'
	import { Button, Dropdown, DropdownItem } from 'flowbite-svelte'
	import Icon from 'svelte-awesome'
	import Editor from '../Editor.svelte'
	import FlowViewer from '../FlowViewer.svelte'
	import Modal from '../Modal.svelte'
	import CollapseLink from './../CollapseLink.svelte'
	import CronInput from './../CronInput.svelte'
	import FlowBox from './../flows/FlowBox.svelte'
	import { flowStore, initFlow } from './../flows/flowStore'
	import Path from './../Path.svelte'
	import Required from './../Required.svelte'
	import SchemaForm from './../SchemaForm.svelte'
	import Toggle from './../Toggle.svelte'
	import Tooltip from './../Tooltip.svelte'
	import { stepOpened } from './stepOpenedStore'
	import { cleanInputs } from './utils'

	export let pathError = ''
	export let initialPath: string = ''

	export let previewArgs: Record<string, any> = {}
	export let scheduleArgs: Record<string, any> = {}
	export let scheduleEnabled = false
	export let scheduleCron: string = '0 */5 * * *'

	let jsonSetter: Modal
	let jsonViewer: Modal
	let jsonValue: string = ''
</script>

<Modal bind:this={jsonSetter}>
	<div slot="title">Import JSON</div>
	<div slot="content" class="h-full">
		<Editor bind:code={jsonValue} lang={'json'} class="h-full" />
	</div>
	<div slot="submission">
		<button
			class="default-button px-4 py-2 font-semibold"
			on:click={() => {
				Object.assign($flowStore, JSON.parse(jsonValue))
				initFlow($flowStore)
				stepOpened.update(() => undefined)
				sendUserToast('OpenFlow imported from JSON')
				jsonSetter.closeModal()
			}}
		>
			Import
		</button>
	</div>
</Modal>

<Modal bind:this={jsonViewer}>
	<div slot="title">See JSON</div>
	<div slot="content" class="h-full">
		<FlowViewer flow={cleanInputs($flowStore)} tab="json" />
	</div>
</Modal>

<FlowBox title="Flow Settings">
	<div slot="header">
		<div class="flex flex-row-reverse">
			<Dropdown class="w-fit" placement="bottom-end">
				<button slot="trigger" class="text-gray-900 bg-white dark:text-white dark:bg-gray-800">
					...
				</button>
				<DropdownItem
					on:click={() => {
						jsonSetter.openModal()
					}}
				>
					<Icon data={faFileImport} scale={0.6} class="inline mr-2" />
					Import from a JSON OpenFlow
				</DropdownItem>
				<DropdownItem
					on:click={() => {
						jsonViewer.openModal()
					}}
				>
					<Icon data={faFileExport} scale={0.6} class="inline mr-2" />
					Export to a JSON OpenFlow
				</DropdownItem>
				<DropdownItem
					on:click={() => {
						const url = new URL('https://hub.windmill.dev/flows/add')
						const openFlow = {
							value: $flowStore.value,
							summary: $flowStore.summary,
							description: $flowStore.description,
							schema: $flowStore.schema
						}
						url.searchParams.append('flow', btoa(JSON.stringify(openFlow)))
						window.open(url, '_blank')?.focus()
					}}
				>
					<Icon data={faGlobe} scale={0.6} class="inline mr-2" />
					Publish to Hub
				</DropdownItem>
			</Dropdown>
		</div>
	</div>

	<div slot="content">
		<div class="p-6 border-t border-gray-300">
			<Path
				bind:error={pathError}
				bind:path={$flowStore.path}
				{initialPath}
				namePlaceholder="my_flow"
				kind="flow"
			>
				<div slot="ownerToolkit">
					Flow permissions depend on their path. Select the group <span class="font-mono">all</span>
					to share your flow, and <span class="font-mono">user</span> to keep it private.
					<a href="https://docs.windmill.dev/docs/reference/namespaces">docs</a>
				</div>
			</Path>

			<label class="block mt-4">
				<span class="text-gray-700">Summary <Required required={false} /></span>
				<textarea
					bind:value={$flowStore.summary}
					class="mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-indigo-300 focus:ring focus:ring-indigo-200 focus:ring-opacity-50"
					placeholder="A very short summary of the flow displayed when the flow is listed"
					rows="1"
				/>
			</label>

			<CollapseLink text="set primary schedule" open={true}>
				<Tooltip>
					The primary schedule of a flow is simply a schedule that has the same name as a flow. It
					can be set and enabled directly within the flow editor. "Watching for new changes" flows
					are meant to be watching regularly for new items in an external systems. The primary
					schedule purpose is there to set the periodicity at which you want this watcher to
					operate.
				</Tooltip>
				<Toggle
					bind:checked={scheduleEnabled}
					options={{
						left: 'disabled',
						right: 'enabled'
					}}
				/>
				<div class="p-2 my-2 rounded" class:bg-gray-300={!scheduleEnabled}>
					{#if !scheduleEnabled}
						<span class="font-black">No next scheduled run when disabled</span>
					{/if}
					<CronInput bind:schedule={scheduleCron} />
				</div>
				<div class="flex flex-row-reverse">
					<Button
						color="alternative"
						size="sm"
						on:click={() => (scheduleArgs = JSON.parse(JSON.stringify(previewArgs)))}
					>
						Copy from preview arguments
					</Button>
				</div>
				<SchemaForm schema={$flowStore.schema} bind:args={scheduleArgs} />
			</CollapseLink>
		</div>
	</div>
</FlowBox>
