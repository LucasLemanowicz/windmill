<script lang="ts">
	import { ScheduleService } from '$lib/gen'

	import { workspaceStore } from '$lib/stores'
	import FlowSettings from './flows/FlowSettings.svelte'
	import { flowStateStore } from './flows/flowState'
	import { flowStore } from './flows/flowStore'
	import FlowTimeline from './flows/FlowTimeline.svelte'

	export let pathError = ''
	export let initialPath: string = ''

	export let scheduleArgs: Record<string, any> = {}
	export let scheduleEnabled = false
	export let scheduleCron: string = '0 */5 * * *'
	export let previewArgs: Record<string, any> = {}

	let scheduleLoaded = false

	async function loadSchedule() {
		if (!scheduleLoaded) {
			scheduleLoaded = true

			const existsSchedule = await ScheduleService.existsSchedule({
				workspace: $workspaceStore ?? '',
				path: initialPath
			})
			if (existsSchedule) {
				const schedule = await ScheduleService.getSchedule({
					workspace: $workspaceStore ?? '',
					path: initialPath
				})

				scheduleEnabled = schedule.enabled!
				scheduleCron = schedule.schedule
				scheduleArgs = schedule.args ?? {}
				previewArgs = JSON.parse(JSON.stringify(scheduleArgs))
			}
		}
	}

	$: if ($flowStore && $workspaceStore && initialPath != '') {
		loadSchedule()
	}
</script>

{#if $flowStateStore}
	<div class="flex space-y-8 flex-col items-center">
		<FlowTimeline bind:args={previewArgs} bind:flowModuleSchemas={$flowStateStore}>
			<div slot="settings">
				<FlowSettings
					bind:pathError
					bind:initialPath
					bind:scheduleArgs
					{previewArgs}
					bind:scheduleCron
					bind:scheduleEnabled
				/>
			</div>
		</FlowTimeline>
	</div>
{:else}
	<h3>Loading flow</h3>
{/if}
