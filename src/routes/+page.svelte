<script lang="ts">
	import axios from 'axios';
	import classNames from 'classnames';
	import { writable } from 'svelte/store';

	let fileInput: HTMLInputElement;

	let participants = writable<string[]>([]);
	let winners = writable<string[]>([]);

	function triggerFileSelect() {
		fileInput.click();
	}

	function handleFileChange(event: Event) {
		const target = event.target as HTMLInputElement;
		const file = target.files?.[0];

		if (!file) return;
		if (file.type !== 'text/csv') {
			alert('Please upload a valid .csv file.');
			return;
		}

		const reader = new FileReader();

		reader.onload = () => {
			const content = reader.result as string;
			const lines = content
				.split('\n')
				.map((line) => line.trim())
				.filter((line) => line);
			participants.set(lines);
		};

		reader.onerror = () => {
			console.error('Error reading file:', reader.error);
		};

		reader.readAsText(file);
	}

	async function fetchUniqueRandomIndices(
		count: number,
		min: number,
		max: number
	): Promise<number[]> {
		const indices = new Set();

		while (indices.size < count) {
			const needed = count - indices.size;
			const url = `https://www.random.org/integers/?num=${needed}&min=${min}&max=${max}&col=1&base=10&format=plain&rnd=new`;

			const response = await axios.get(url);
			let { data } = response;

			let numbers;
			if (typeof data === 'string') {
				numbers = data
					.split('\n')
					.map((n) => parseInt(n, 10))
					.filter((n) => !Number.isNaN(n));
			} else {
				numbers = [data];
			}

			numbers.forEach((n) => indices.add(n));
		}

		return Array.from(indices) as number[];
	}

	async function rollRandomWinners() {
		const participantCount = $participants.length;
		if (participantCount <= 0) {
			alert('No participants to roll from.');
			return;
		}
		if (winnerCount <= 0 || winnerCount > participantCount) {
			alert('Invalid winner count. Please enter a number between 1 and ' + participantCount);
			return;
		}
		const indexes = await fetchUniqueRandomIndices(winnerCount, 0, participantCount - 1);
		winners.set(indexes.map((index) => $participants[index]));
	}

	let winnerCount = 1;

	function downloadCSV(content: string, filename = 'data.csv') {
		const blob = new Blob([content], { type: 'text/csv;charset=utf-8;' });
		const url = URL.createObjectURL(blob);

		const link = document.createElement('a');
		link.setAttribute('href', url);
		link.setAttribute('download', filename);
		link.style.display = 'none';

		document.body.appendChild(link);
		link.click();
		document.body.removeChild(link);
		URL.revokeObjectURL(url); // Clean up
	}
</script>

<div class="grid h-screen w-screen grid-cols-2 gap-6 p-6">
	<button on:click={triggerFileSelect} class="btn btn-primary btn-lg col-span-2"
		>Load participants CSV</button
	>

	<input
		class="hidden"
		type="file"
		accept=".csv"
		bind:this={fileInput}
		on:change={handleFileChange}
	/>
	<div class="h-full overflow-hidden rounded-lg bg-black/40">
		<div class="h-full w-full overflow-y-auto">
			<table class="table w-full">
				<thead>
					<tr>
						<th>Participant</th>
						<th class="text-right">Total: {$participants.length} </th>
					</tr>
				</thead>
				<tbody>
					{#each $participants as participant, i}
						<tr>
							<td>{i}</td>
							<td>{participant}</td>
						</tr>
					{/each}
				</tbody>
			</table>
		</div>
	</div>
	<div
		class={classNames(
			'overflow-hidden rounded-lg bg-black/40',
			$winners.length ? 'opacity-100' : 'opacity-50'
		)}
	>
		<div class="h-full w-full overflow-y-auto">
			<table class="table w-full">
				<thead>
					<tr>
						<th>Winners</th>
						<th class="text-right">Total: {$winners.length} </th>
					</tr>
				</thead>
				<tbody>
					{#each $winners as winner, i}
						<tr>
							<td>{i}</td>
							<td>{winner}</td>
						</tr>
					{/each}
				</tbody>
			</table>
		</div>
	</div>
	<div class="grid h-min grid-cols-2">
		<label class="input">
			Winner Count
			<input
				disabled={$participants.length <= 0}
				class="text-right"
				type="number"
				placeholder="Winner Count"
				bind:value={winnerCount}
			/>
		</label>

		<button
			disabled={$participants.length <= 0}
			on:click={rollRandomWinners}
			class="btn btn-primary">Roll Winners</button
		>
	</div>

	<button
		disabled={$winners.length <= 0}
		class="btn btn-primary"
		on:click={() => downloadCSV($winners.join('\n'), `winners-${new Date().toLocaleString()}.csv`)}
	>
		Download Winners CSV
	</button>
</div>
