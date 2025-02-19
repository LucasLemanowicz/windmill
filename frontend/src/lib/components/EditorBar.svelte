<script lang="ts">
	import { ResourceService, ScriptService, VariableService } from '$lib/gen'
	import { getScriptByPath, loadHubScripts, sendUserToast } from '$lib/utils'

	import { faSearch } from '@fortawesome/free-solid-svg-icons'
	import Icon from 'svelte-awesome'

	import { hubScripts, workspaceStore } from '$lib/stores'
	import { Button } from 'flowbite-svelte'
	import { Highlight } from 'svelte-highlight'
	import { python, typescript } from 'svelte-highlight/languages'
	import type Editor from './Editor.svelte'
	import ItemPicker from './ItemPicker.svelte'
	import Modal from './Modal.svelte'
	import ResourceEditor from './ResourceEditor.svelte'
	import VariableEditor from './VariableEditor.svelte'

	export let lang: 'python3' | 'deno'
	export let editor: Editor
	export let websocketAlive: { pyright: boolean; black: boolean; deno: boolean }

	let variablePicker: ItemPicker
	let resourcePicker: ItemPicker
	let scriptPicker: ItemPicker
	let variableEditor: VariableEditor
	let resourceEditor: ResourceEditor

	let codeViewer: Modal
	let codeLang: 'python3' | 'deno' = 'deno'
	let codeContent: string = ''

	async function loadVariables() {
		let r: { name: string; path?: string; description?: string }[] = []
		const variables = (
			await VariableService.listVariable({ workspace: $workspaceStore ?? 'NO_W' })
		).map((x) => {
			return { name: x.path, ...x }
		})

		const rvariables = await VariableService.listContextualVariables({
			workspace: $workspaceStore ?? 'NO_W'
		})
		r = r.concat(variables).concat(rvariables)
		return r
	}

	async function loadScripts(): Promise<{ path: string; summary?: string }[]> {
		const workspaceScripts: { path: string; summary?: string }[] = await ScriptService.listScripts({
			workspace: $workspaceStore ?? 'NO_W'
		})
		await loadHubScripts()
		const hubScripts_ = $hubScripts ?? []

		return workspaceScripts.concat(hubScripts_)
	}
</script>

<ItemPicker
	bind:this={scriptPicker}
	pickCallback={async (path, _) => {
		const { language, content } = await getScriptByPath(path ?? '')
		codeContent = content
		codeLang = language
		codeViewer.openModal()
	}}
	closeOnClick={false}
	itemName="script"
	extraField="summary"
	loadItems={loadScripts}
/>

<Modal bind:this={codeViewer}>
	<div slot="title">Code</div>
	<div slot="content">
		{#if codeLang == 'python3'}
			<Highlight language={python} code={codeContent} />
		{:else if codeLang == 'deno'}
			<Highlight language={typescript} code={codeContent} />
		{/if}
	</div></Modal
>

<ItemPicker
	bind:this={variablePicker}
	pickCallback={(path, name) => {
		if (!path) {
			if (lang == 'deno') {
				editor.insertAtCursor(`Deno.env.get('${name}')`)
			} else {
				if (!editor.getCode().includes('import os')) {
					editor.insertAtBeginning('import os\n')
				}
				editor.insertAtCursor(`os.environ.get("${name}")`)
			}
			sendUserToast(`${name} inserted at cursor`)
		} else {
			if (lang == 'deno') {
				if (!editor.getCode().includes('import * as wmill from')) {
					editor.insertAtBeginning(
						`import * as wmill from 'https://deno.land/x/windmill@v${__pkg__.version}/mod.ts'\n`
					)
				}
				editor.insertAtCursor(`(await wmill.getVariable('${path}'))`)
			} else {
				if (!editor.getCode().includes('import wmill')) {
					editor.insertAtBeginning('import wmill\n')
				}
				editor.insertAtCursor(`wmill.get_variable("${path}")`)
			}
			sendUserToast(`${name} inserted at cursor`)
		}
	}}
	itemName="Variable"
	extraField="name"
	loadItems={loadVariables}
>
	<div slot="submission" class="flex flex-row">
		<div class="text-xs mr-2 align-middle">
			The variable you were looking for does not exist yet?
		</div>
		<button
			class="default-button-secondary"
			type="button"
			on:click={() => {
				variableEditor.initNew()
			}}
		>
			Create a new variable
		</button>
	</div>
</ItemPicker>

<ItemPicker
	bind:this={resourcePicker}
	pickCallback={(path, _) => {
		if (lang == 'deno') {
			if (!editor.getCode().includes('import * as wmill from')) {
				editor.insertAtBeginning(
					`import * as wmill from 'https://deno.land/x/windmill@v${__pkg__.version}/mod.ts'\n`
				)
			}
			editor.insertAtCursor(`(await wmill.getResource('${path}'))`)
		} else {
			if (!editor.getCode().includes('import wmill')) {
				editor.insertAtBeginning('import wmill\n')
			}
			editor.insertAtCursor(`wmill.get_resource("${path}")`)
		}
		sendUserToast(`${path} inserted at cursor`)
	}}
	itemName="Resource"
	extraField="resource_type"
	loadItems={async () =>
		await ResourceService.listResource({ workspace: $workspaceStore ?? 'NO_W' })}
>
	<div slot="submission" class="flex flex-row">
		<div class="text-xs mr-2 align-middle">
			The resource you were looking for does not exist yet?
		</div>
		<button
			class="default-button-secondary"
			type="button"
			on:click={() => {
				resourceEditor.initNew()
			}}
		>
			Create a new resource
		</button>
	</div>
</ItemPicker>

<ResourceEditor bind:this={resourceEditor} on:refresh={resourcePicker.openModal} />

<VariableEditor bind:this={variableEditor} on:create={variablePicker.openModal} />

<div class="flex flex-row w-full">
	<div class="flex flex-row w-full space-x-2 whitespace-nowrap">
		<Button
			size="xs"
			color="alternative"
			on:click={() => {
				variablePicker.openModal()
			}}
		>
			<Icon data={faSearch} scale={0.7} class="mr-2" />
			Variable picker
		</Button>

		<Button
			size="xs"
			color="alternative"
			on:click={() => {
				resourcePicker.openModal()
			}}
		>
			<Icon data={faSearch} scale={0.7} class="mr-2" />
			Resource picker
		</Button>

		<Button
			size="xs"
			color="alternative"
			on:click={() => {
				scriptPicker.openModal()
			}}
		>
			<Icon data={faSearch} scale={0.7} class="mr-2" />
			Script explorer
		</Button>
	</div>
	<div class="flex flex-row-reverse gap-x-2 w-full">
		<Button
			size="xs"
			class={!websocketAlive.pyright && !websocketAlive.deno && !websocketAlive.black
				? 'bg-red-50'
				: ''}
			color="alternative"
			on:click={() => {
				editor.reloadWebsocket()
			}}
		>
			Reload assistants
			<span class="ml-1">
				{#if lang == 'deno'}
					(<span class={websocketAlive.deno ? 'text-green-600' : 'text-red-700'}>Deno</span>)
				{:else if lang == 'python3'}
					(<span class={websocketAlive.pyright ? 'text-green-600' : 'text-red-700'}>Pyright</span>
					&nbsp;
					<span class={websocketAlive.black ? 'text-green-600' : 'text-red-700'}>Black</span>)
				{/if}
			</span>
		</Button>
		<Button
			size="xs"
			color="alternative"
			on:click={() => {
				editor.clearContent()
			}}>Reset content</Button
		>
	</div>
</div>
