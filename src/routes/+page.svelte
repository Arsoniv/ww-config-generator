<script lang="ts">
	type res = {
		width: number;
		height: number;
		keybind: string;
		modifyers: boolean;
		mods?: string[];
		note?: string;
	};

	type box = {
		width: number;
		height: number;
		x: number;
		y: number;
	};

	type mirror = {
		src: box;
		dst: box;
	};

	let keyboard_layout = $state('');
	let repeat_rate = $state(30);
	let repeat_delay = $state(300);
	let sens = $state(1.0);
	let confine_pointer = $state(false);
	let background_color = $state('#000000');
	let ninB_pos = $state('right');
	let ninB_opacity = $state(0.7);
	let ninB_toggle = $state('B');
	let ninB_path = $state('ninb');

	let resoloutions: res[] = $state([
		{ width: 300, height: 850, keybind: 'G', modifyers: false, note: 'Thin' },
		{ width: 1920, height: 300, keybind: 'F', modifyers: false, note: 'Wide' },
		{
			width: 250,
			height: 16384,
			keybind: 'P',
			modifyers: true,
			mods: ['ctrl'],
			note: 'Tall (measureing)'
		}
	]);

	let mirrors: mirror[] = $derived([
		{
			src: { width: 120, height: 100, x: 0, y: 0 },
			dst: { width: 240, height: 200, x: 100, y: 100 }
		},
		{
			src: { width: 1280, height: 720, x: 0, y: 0 },
			dst: { width: 800, height: 600, x: 200, y: 150 }
		}
	]);

	function addResolution() {
		resoloutions = [
			...resoloutions,
			{ width: 800, height: 600, keybind: '', modifyers: false, mods: [] }
		];
	}

	function removeResolution(index: number) {
		resoloutions = resoloutions.filter((_, i) => i !== index);
	}

	function addMirror() {
		mirrors = [
			...mirrors,
			{
				src: { width: 120, height: 100, x: 0, y: 0 },
				dst: { width: 240, height: 200, x: 100, y: 100 }
			}
		];
	}

	function removeMirror(index: number) {
		mirrors = mirrors.filter((_, i) => i !== index);
	}

	function toggleModifier(index: number, mod: string) {
		if (!resoloutions[index].mods) resoloutions[index].mods = [];
		const mods = resoloutions[index].mods!;
		const modIndex = mods.indexOf(mod);
		if (modIndex === -1) mods.push(mod);
		else mods.splice(modIndex, 1);
	}

	let resolutionsLua = $derived(
		resoloutions
			.map((r, i) => `  res${i + 1} = helpers.toggle_res{ ${r.width}, ${r.height} },`)
			.join('\n')
	);

	let resolutionsBindsLua = $derived(
		resoloutions
			.map((r, i) => {
				if (r.modifyers) {
					let modsString = r.mods?.join('-') ?? '';
					let fullKey = modsString ? `${modsString}-${r.keybind}` : r.keybind;
					return `  ["${fullKey}"] = resolutions.res${i + 1},`;
				} else {
					return `  ["*-${r.keybind}"] = resolutions.res${i + 1},`;
				}
			})
			.join('\n')
	);

	let mirrorsLua = $derived(
		mirrors
			.map(
				(r) =>
					`
waywall.mirror(
  {
    src = { x = ${r.src.x}, y = ${r.src.y}, w = ${r.src.width}, h = ${r.src.height} },
    dst = { x = ${r.dst.x}, y = ${r.dst.y}, w = ${r.dst.width}, h = ${r.dst.height} },
  },
)`
			)
			.join('\n')
	);

	let text = $derived(`
local waywall = require("waywall")
local helpers = require("waywall.helpers")

local config = {
  input = {
    layout = "${keyboard_layout}",
    repeat_rate = ${repeat_rate},
    repeat_delay = ${repeat_delay},

    sensitivity = ${sens},
    confine_pointer = ${confine_pointer},
  },
  theme = {
    background = "${background_color}",
    ninb_anchor = "${ninB_pos}",
    ninb_opacity = ${ninB_opacity},
  },
}
${mirrorsLua}

local resolutions = {
${resolutionsLua}
}

local exec_ninb = function()
    waywall.exec("${ninB_path}")
end

config.actions = {

  ["*-${ninB_toggle}"] = helpers.toggle_floating,

${resolutionsBindsLua}
}

return config
`);

	let copied = $state(false);
	async function copy() {
		await navigator.clipboard.writeText(text);
		copied = true;
		setTimeout(() => (copied = false), 2000);
	}
</script>

<div class="mx-auto mt-6 mb-6 max-w-5xl rounded-lg bg-neutral-900 p-6 font-sans text-white">
	<h2 class="mb-4 text-2xl font-semibold">Settings</h2>
	<div class="mb-6 grid grid-cols-1 gap-4 sm:grid-cols-2">
		<label class="flex flex-col">
			Keyboard Layout
			<input
				type="text"
				bind:value={keyboard_layout}
				class="mt-1 rounded border border-neutral-700 bg-neutral-800 p-2"
			/>
		</label>
		<label class="flex flex-col">
			Repeat Rate
			<input
				type="number"
				bind:value={repeat_rate}
				min="1"
				class="mt-1 rounded border border-neutral-700 bg-neutral-800 p-2"
			/>
		</label>
		<label class="flex flex-col">
			Repeat Delay (ms)
			<input
				type="number"
				bind:value={repeat_delay}
				min="0"
				class="mt-1 rounded border border-neutral-700 bg-neutral-800 p-2"
			/>
		</label>
		<label class="flex flex-col">
			Sensitivity
			<input
				type="number"
				step="0.1"
				bind:value={sens}
				min="0"
				class="mt-1 rounded border border-neutral-700 bg-neutral-800 p-2"
			/>
		</label>
		<label class="flex flex-col">
			Ninbot Position
			<select
				bind:value={ninB_pos}
				class="mt-1 rounded border border-neutral-700 bg-neutral-800 p-2"
			>
				<option value="topleft">Top Left</option>
				<option value="top">Top</option>
				<option value="topright">Top Right</option>
				<option value="left">Left</option>
				<option value="right">Right</option>
				<option value="bottomleft">Bottom Left</option>
				<option value="bottomright">Bottom Right</option>
			</select>
		</label>
		<label class="flex flex-col">
			Ninbot Toggle <input
				class="mt-1 rounded border border-neutral-700 bg-neutral-800 p-2 uppercase"
				type="text"
				bind:value={ninB_toggle}
			/></label
		>
		<label class="flex items-center space-x-2">
			<span>Confine Pointer To Window</span>
			<input type="checkbox" bind:checked={confine_pointer} />
		</label>
		<label class="flex space-x-2">
			<span>Background Color</span>
			<input
				type="color"
				bind:value={background_color}
				class="mt-1 rounded border border-neutral-700 p-1"
			/>
		</label>
		<label class="flex flex-col">
			Ninbot Opacity: {ninB_opacity.toFixed(2)}
			<input
				type="range"
				min="0"
				max="1"
				step="0.05"
				bind:value={ninB_opacity}
				class="mt-1 w-full"
			/>
		</label>
		<label class="flex flex-col">
			Ninbot Path
			<input
				type="text"
				bind:value={ninB_path}
				class="mt-1 rounded border border-neutral-700 bg-neutral-800 p-2"
			/>
		</label>
	</div>

	<h2 class="mb-4 text-2xl font-semibold">Resolutions</h2>
	<div class="mb-4 max-h-96 space-y-4 overflow-y-auto">
		{#each resoloutions as res, i (i)}
			<div
				class="flex flex-col gap-3 rounded border border-neutral-700 bg-neutral-800 p-4 sm:flex-row sm:items-center sm:space-x-4"
			>
				<div class="flex flex-grow flex-wrap gap-2">
					<label class="flex flex-col text-sm">
						Width
						<input
							type="number"
							bind:value={res.width}
							class="w-20 rounded border border-neutral-700 bg-neutral-900 p-1"
							min="0"
						/>
					</label>
					<label class="flex flex-col text-sm">
						Height
						<input
							type="number"
							bind:value={res.height}
							class="w-20 rounded border border-neutral-700 bg-neutral-900 p-1"
							min="0"
						/>
					</label>
					<label class="flex flex-col text-sm">
						Keybind
						<input
							type="text"
							bind:value={res.keybind}
							class="w-16 rounded border border-neutral-700 bg-neutral-900 p-1 text-center uppercase"
						/>
					</label>
				</div>
				{#if res.note}
					<h2>{res.note}</h2>
				{/if}
				<div class="flex min-w-[150px] flex-col gap-1 text-sm">
					<span
						>Modifiers
						<input type="checkbox" bind:checked={res.modifyers} />
					</span>
					<div class="flex flex-wrap gap-2">
						{#each ['ctrl', 'shift', 'alt'] as mod}
							<label class="inline-flex cursor-pointer items-center select-none">
								<span class="capitalize"
									>{mod}
									<input
										type="checkbox"
										checked={res.mods?.includes(mod)}
										onchange={() => toggleModifier(i, mod)}
										class="mr-1"
										disabled={!res.modifyers}
									/></span
								>
							</label>
						{/each}
					</div>
				</div>
				<button
					onclick={() => removeResolution(i)}
					class="self-start font-semibold text-red-500 hover:text-red-600"
					aria-label="Remove resolution"
				>
					✕
				</button>
			</div>
		{/each}
	</div>

	<button
		onclick={addResolution}
		class="rounded bg-green-600 px-4 py-2 font-semibold text-white hover:bg-green-700"
	>
		+ Add Resolution
	</button>
	<h2 class="mt-4 mb-4 text-2xl font-semibold">Mirrors</h2>
	<div class="mb-4 max-h-96 space-y-4 overflow-y-auto">
		{#each mirrors as mirror, i (i)}
			<div
				class="flex flex-col gap-3 rounded border border-neutral-700 bg-neutral-800 p-4 sm:flex-row sm:items-center sm:space-x-4"
			>
				<div class="flex flex-grow flex-wrap gap-2">
					<h2 class="m-2 mt-4 flex flex-col text-sm font-bold">Source:</h2>
					<label class="flex flex-col text-sm">
						Width
						<input
							type="number"
							bind:value={mirror.src.width}
							class="w-20 rounded border border-neutral-700 bg-neutral-900 p-1"
							min="0"
						/>
					</label>
					<label class="flex flex-col text-sm">
						Height
						<input
							type="number"
							bind:value={mirror.src.height}
							class="w-20 rounded border border-neutral-700 bg-neutral-900 p-1"
							min="0"
						/>
					</label>
					<label class="flex flex-col text-sm">
						X
						<input
							type="number"
							bind:value={mirror.src.x}
							class="w-20 rounded border border-neutral-700 bg-neutral-900 p-1"
						/>
					</label>
					<label class="flex flex-col text-sm">
						Y
						<input
							type="number"
							bind:value={mirror.src.y}
							class="w-20 rounded border border-neutral-700 bg-neutral-900 p-1"
						/>
					</label>
				</div>
				<div class="flex flex-grow flex-wrap gap-2">
					<h2 class="m-2 mt-4 flex flex-col text-sm font-bold">Destination:</h2>
					<label class="flex flex-col text-sm">
						Width
						<input
							type="number"
							bind:value={mirror.dst.width}
							class="w-20 rounded border border-neutral-700 bg-neutral-900 p-1"
							min="0"
						/>
					</label>
					<label class="flex flex-col text-sm">
						Height
						<input
							type="number"
							bind:value={mirror.dst.height}
							class="w-20 rounded border border-neutral-700 bg-neutral-900 p-1"
							min="0"
						/>
					</label>
					<label class="flex flex-col text-sm">
						X
						<input
							type="number"
							bind:value={mirror.dst.x}
							class="w-20 rounded border border-neutral-700 bg-neutral-900 p-1"
						/>
					</label>
					<label class="flex flex-col text-sm">
						Y
						<input
							type="number"
							bind:value={mirror.dst.y}
							class="w-20 rounded border border-neutral-700 bg-neutral-900 p-1"
						/>
					</label>
				</div>
				<button
					onclick={() => removeMirror(i)}
					class="self-start font-semibold text-red-500 hover:text-red-600"
					aria-label="Remove mirror"
				>
					✕
				</button>
			</div>
		{/each}
	</div>

	<button
		onclick={addMirror}
		class="rounded bg-green-600 px-4 py-2 font-semibold text-white hover:bg-green-700"
	>
		+ Add Mirror
	</button>
</div>

<div class="mx-auto mb-6 max-w-5xl rounded-lg bg-neutral-900 p-6 font-sans text-white">
	<button
		class="rounded border-2 border-neutral-600 bg-neutral-800 px-4 py-2 text-white transition hover:bg-neutral-700"
		onclick={copy}
	>
		{copied ? 'Copied!' : 'Copy'}
	</button>

	<pre
		class="m-10 mt-5 flex rounded bg-neutral-800 p-10 font-mono text-base whitespace-pre-wrap text-white sm:text-lg">
    <code>{text}</code>
  </pre>
</div>
