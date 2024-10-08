<script lang="ts">
	import type { Header } from '$lib/dat-viewer/headers';
	import type { BundleIndex } from '$lib/patchcdn/index-store';
	import { TreeView, TreeViewItem } from '@skeletonlabs/skeleton';
	import type { DatFile } from 'pathofexile-dat/dat.js';
	import { onMount } from 'svelte';

	let loader;
	let index: BundleIndex;
	let db: any;
	let dirs: string[] = [];
	let dirContents: { files: string[]; dirs: string[] } = { files: [], dirs: [] };
	let fileContent: Uint8Array | null = null;
	let headers: Header[] = []; // Store headers here
	let currentPath: string = '';
	export let rows: any[] = []; // To store rows from the .dat file

	// Load directory contents
	async function loadDirContents(dirPath: string) {
		currentPath = dirPath; // Update current path

		const contents = index.getDirContent(dirPath) || { files: [], dirs: [] };
		dirContents = { dirs: contents.dirs, files: contents.files };
	}

	// Load file content
	async function loadFileContent(filePath: string) {
		fileContent = await index.loadFileContent(filePath); // Load file content

		// Extract schema name
		const schemaName = filePath.replace('data/', '').replace('.dat64', '');

		// Dynamically import necessary functions
		const { readDatFile, readColumn } = await import('pathofexile-dat/dat.js');
		const { analyzeDatFile } = await import('$lib/worker/interface');
		const { fromSerializedHeaders } = await import('$lib/dat-viewer/headers');

		// Parse `.dat64` file
		const datFile = readDatFile(filePath, fileContent);
		const columnStats = await analyzeDatFile(datFile);

		// Find and apply headers
		const serializedHeaders = await db.findByName(schemaName);
		const headersResult = fromSerializedHeaders(serializedHeaders, columnStats, datFile);

		if (headersResult) {
			headers = headersResult.headers;
			console.log('Loaded file headers:', headers);

			// Time the operation of extracting rows
			const startTime = performance.now();

			// Extract rows from the datFile using the headers
			rows = await extractRows(datFile, headers);
			console.log('Extracted rows:', rows);

			const endTime = performance.now();
			console.log(`Extracting rows took ${endTime - startTime} milliseconds`);
		} else {
			console.warn(`Invalid headers for file: ${schemaName}`);
		}
	}


</script>


<section class="container">
  				<!-- Display the headers in a table -->
				{#if headers.length > 0}
					<h3>File Headers</h3>
					<table border="1" style="width: 100%;">
						<thead>
							<tr>
								<th>Name</th>
								<th>Offset</th>
								<th>Length</th>
								<th>Type</th>
							</tr>
						</thead>
						<tbody>
							{#each headers as header}
								<tr>
									<td>{header.name ? header.name : 'Unnamed'}</td>
									<td>{header.offset}</td>
									<td>{header.length}</td>
									<td>
										{#if header.type.integer}
											Integer (Size: {header.type.integer.size}, Unsigned: {header.type.integer
												.unsigned})
										{:else if header.type.string}
											String
										{:else if header.type.decimal}
											Decimal (Size: {header.type.decimal.size})
										{:else if header.type.boolean}
											Boolean
										{:else if header.type.key}
											Key (Foreign: {header.type.key.foreign}, Table: {header.type.key.table ||
												'None'})
										{/if}
									</td>
								</tr>
							{/each}
						</tbody>
					</table>
				{/if}

				<!-- Display the data rows -->
				{#if rows.length > 0}
					<h3>Data Rows</h3>
					<table border="1" style="width: 100%;">
						<thead>
							<tr>
								{#each headers as header}
									<th>{header.name ? header.name : 'Unnamed'}</th>
								{/each}
							</tr>
						</thead>
						<tbody>
							{#each rows as row}
								<tr>
									{#each Object.keys(row) as key}
										<td>{row[key]}</td>
									{/each}
								</tr>
							{/each}
						</tbody>
					</table>
				{/if}
</section>