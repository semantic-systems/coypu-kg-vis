<script lang="ts">
	import { ChevronDown, FolderOpen, Plus, XMark } from '@steeze-ui/heroicons';
	import { Icon } from '@steeze-ui/svelte-icon';
	import isUrl from 'is-url';
	import { SPARQL, type BindingContent } from '../../../api/sparql';
	import { CurrentGraph, type Node, type URI } from '../../../api/graph';
	import { getContext, onDestroy, onMount } from 'svelte';
	import LoadingCircle from '../../util/Loading-Circle.svelte';
	import { Settings } from '../../../settings';
	import { Button, Chevron, Dropdown, DropdownItem, Hr } from 'flowbite-svelte';
	import { click_outside, scrollbar_width } from '../../../util';
	import { ModalManager } from '../modal-store';
	import { LoaderManager } from '../../../components/loader/graph-loader';

	let error: string = '';
	let loading: boolean = false;
	let nodes: Node[] = [];
	let sparql_query_area: HTMLTextAreaElement;
	let sparql_query: string = '';
	let dropdown_open: boolean = false;
	const query_examples: { name: string; query: string }[] = [
		{
			name: 'Dogs',
			query: `SELECT ?dog 
WHERE 
{
	?dog wdt:P31 wd:Q144. 
} LIMIT 100`
		},
		{
			name: 'Cats',
			query: `SELECT ?cat 
WHERE 
{
	?cat wdt:P31 wd:Q146. 
} LIMIT 100`
		}
	];

	function select_query_example(example: { name: string; query: string }) {
		sparql_query = example.query;
		$Settings.advanced_settings = true;
		dropdown_open = false;
		setTimeout(update_height, 0);
	}

	function close() {
		ModalManager.close();
	}

	function submit_query() {
		if (loading) return;
		loading = true;
		SPARQL.query(sparql_query)
			.then(async (results) => {
				error = '';
				if (results.length === 0) {
					loading = false;
					return;
				}
				const all_nodes = results
					.flatMap((b) => Object.values(b))
					.filter((content) => content) as Required<BindingContent>[];
				const literal_nodes = all_nodes
					.filter((node) => node.type === 'literal')
					.map((node) => {
						const new_node = $CurrentGraph.find_or_create_node(
							node.value,
							node.value,
							'literal',
							false
						);
						node.datatype && (new_node.datatype = node.datatype);
						node['xml:lang'] && (new_node.language = node['xml:lang']);
						return new_node;
					});

				const entity_nodes = await $CurrentGraph.load_nodes(
					all_nodes.filter((node) => node.type === 'uri').map((node) => node.value),
					false
				);

				nodes = nodes.concat(entity_nodes).concat(literal_nodes);
				loading = false;
			})
			.catch((e) => {
				loading = false;
				console.log(e.message);
				error = e.message;
			});
	}

	function toggle_advanced() {
		$Settings.advanced_settings = !$Settings.advanced_settings;
	}

	let sparql_query_area_height = 0;
	$: {
		sparql_query;
		update_height();
	}

	function update_height() {
		if (sparql_query_area) {
			sparql_query_area.style.height = 'auto';
			sparql_query_area_height = sparql_query_area.scrollHeight;
			sparql_query_area.style.height = sparql_query_area_height + 'px';
		}
	}

	async function fetch_url(url: URI) {
		if (loading) return;
		if (!isUrl(url)) {
			error = 'Invalid URL';
			return;
		}
		if (nodes.find((node) => node.id === url)) {
			error = 'Node already exists';
			return;
		}
		error = '';
		loading = true;
		const node = await $CurrentGraph.load_node(url);
		/* if (label === url) {
			error = 'No label found';
			return;
		} */
		nodes.push(node);
		nodes = nodes;
		loading = false;
	}

	function remove_node(node: Node) {
		$CurrentGraph.nodes = $CurrentGraph.nodes.filter((n) => n !== node);
		nodes = nodes.filter((n) => n !== node);
	}

	async function add_nodes() {
		LoaderManager.open();
		$CurrentGraph.show_nodes(nodes, false);
		$CurrentGraph.load_relations(nodes);
		close();
	}

	// smart search
	type Suggestion = {
		label: string;
		type: string;
		uri: string;
		height?: number;
		suggestion_type: 'direct' | 'search';
		button?: HTMLButtonElement;
	};

	let suggestions: Suggestion[] = [];

	let show_suggestsions = false;

	let entity_url: string = '';
	let debouncer: NodeJS.Timeout;

	// debounce the entity input
	$: {
		suggestions = [];
		if (debouncer) clearTimeout(debouncer);
		debouncer = setTimeout(() => {
			smart_search(entity_url);
		}, 500);
	}

	async function smart_search(input: string) {
		if (!input || input.length === 0) return;
		loading = true;

		let suggestion_promises: Promise<Suggestion[]> | undefined = undefined;
		const url_promises: Promise<Suggestion>[] = [];
		const urls: string[] = [];
		if (isUrl(input)) {
			urls.push(input);

			if (input.startsWith('https://')) {
				input = input.replace('https://', 'http://');
			}

			if ($Settings.endpoint_type === 'wikidata') {
				if (input.includes('/wiki/')) {
					if (input.includes('wiki/Property:')) {
						//urls.push(input.replace('wiki/Property:', 'prop/direct/'));
					} else {
						urls.push(input.replace('/wiki/', '/entity/'));
					}
				}
			} else if ($Settings.endpoint_type === 'dbpedia') {
				if (input.includes('/page/')) {
					urls.push(input.replace('/page/', '/resource/'));
				}
			}
		} else {
			if ($Settings.endpoint_type === 'wikidata') {
				// regex that filters for QIDs e.g. Q123456
				const qid_regex = /Q\d+/g;
				const qids = input.match(qid_regex);
				if (qids) {
					urls.push(...qids.map((qid) => `http://www.wikidata.org/entity/${qid}`));
				}
			} else if ($Settings.endpoint_type === 'dbpedia') {
				urls.push(`http://dbpedia.org/resource/${input}`);
			}

			if ($Settings.smart_search) {
				suggestion_promises = SPARQL.search(input).then((entities) =>
					entities.map((e) => ({
						label: e.label,
						type: e.type,
						uri: e.uri,
						suggestion_type: 'search'
					}))
				);
			}
		}

		url_promises.push(
			...urls.map((u) => {
				u = u.replaceAll(' ', '_');
				return SPARQL.fetch_info(u).then(
					(i) => ({ label: i.label, type: i.type, uri: u, suggestion_type: 'direct' } as Suggestion)
				);
			})
		);

		show_suggestsions = true;
		suggestions = [];
		const [suggestion, ...url] = await Promise.all([suggestion_promises, ...url_promises]);
		if (suggestion) {
			suggestions = [...suggestion.filter((s) => s.label.length > 0 && s.label !== s.uri)];
		}
		suggestions.push(...url.filter((s) => s.label.length > 0 && s.label !== s.uri));
		//filter duplicates based on url
		suggestions = suggestions.filter((s, i, a) => a.findIndex((t) => t.uri === s.uri) === i);
		loading = false;
	}

	let search_input: HTMLInputElement;
	let selected_suggestion: number = -1;

	onMount(() => {
		update_height();
		document.addEventListener('keydown', handle_keydown);
		setTimeout(() => search_input?.focus(), 10);
	});

	onDestroy(() => {
		document.removeEventListener('keydown', handle_keydown);
	});

	function handle_keydown(event: KeyboardEvent) {
		if (event.key === 'Escape') {
			close();
		} else if (event.key === 'ArrowDown') {
			if (suggestions.length > 0) {
				select_suggestion(1);
			}
		} else if (event.key === 'ArrowUp') {
			if (suggestions.length > 0) {
				select_suggestion(-1);
			}
		}
	}

	function select_suggestion(dir: -1 | 1) {
		if (suggestions.length > 0) {
			if (selected_suggestion + dir >= -2) {
				selected_suggestion += dir;
			}

			if (selected_suggestion >= suggestions.length) {
				selected_suggestion = -1;
			} else if (selected_suggestion === -2) {
				selected_suggestion = suggestions.length - 1;
			}

			if (selected_suggestion === -1) {
				search_input.focus();
			} else {
				suggestions[selected_suggestion].button?.focus();
			}
		}
	}
</script>

<div class="flex flex-col sm:w-[475px] w-80">
	<div class="flex justify-between items-center text-center mb-2">
		<h1 class="text-lg font-bold">Add Nodes</h1>
		<div>
			<Button color="light"><Icon src={FolderOpen} size="20" class="mr-2" />Examples</Button>
			<Dropdown frameClass="[&_ul]:!w-32" bind:open={dropdown_open}>
				{#each query_examples as example}
					<DropdownItem on:click={() => select_query_example(example)}>{example.name}</DropdownItem>
				{/each}
			</Dropdown>
		</div>
	</div>

	<button
		class="flex justify-center items-center text-dark-muted dark:text-light-muted pb-1 relative"
		on:click={toggle_advanced}
	>
		Advanced <Icon
			src={ChevronDown}
			class="w-5 h-5 absolute right-0 transition-all duration-200 {!$Settings.advanced_settings
				? 'rotate-0'
				: '-rotate-180'}"
		/>
	</button>
	<Hr divClass="mb-2" />
	<div
		style={`height: ${
			$Settings.advanced_settings && sparql_query_area
				? Math.min(sparql_query_area_height, 350) + 113
				: 0
		}px`}
		class="h-0 overflow-hidden transition-all duration-200 flex gap-2 flex-col mb-2"
	>
		<div
			class="mt-2 w-full border border-gray-200 rounded-lg bg-gray-50 dark:bg-gray-700 dark:border-gray-600"
		>
			<div class="px-4 py-2 bg-white rounded-t-lg dark:bg-gray-800">
				<label for="sparql-query" class="block mb-2 text-sm font-medium">SPARQL Query</label>
				<textarea
					data-gramm="false"
					bind:this={sparql_query_area}
					bind:value={sparql_query}
					id="sparql-query"
					class="resize-none w-full px-0 text-sm text-gray-900 bg-white border-0 dark:bg-gray-800 focus:ring-0 dark:text-light dark:placeholder-gray-400 max-h-[350px]"
					placeholder={`Enter a SPARQL query or select an example.`}
					required
				/>
			</div>
			<div class="flex items-center justify-between px-3 py-2 border-t dark:border-gray-600">
				<Button on:click={submit_query} type="submit" size="sm" disabled={sparql_query.length == 0}
					>Fetch</Button
				>
			</div>
		</div>
	</div>

	<div class="flex gap-2 flex-col mb-2">
		<div class="relative">
			<input
				on:click={() => {
					if (suggestions.length > 0) {
						show_suggestsions = true;
					}
				}}
				bind:this={search_input}
				bind:value={entity_url}
				type="text"
				id="entity_url"
				class="bg-gray-50 border border-gray-300 text-sm rounded-lg focus:ring-primary focus:border-primring-primary block w-full p-2.5 dark:bg-gray-700 dark:border-gray-600  dark:focus:ring-primary dark:focus:border-primring-primary"
				placeholder="Search {$Settings.endpoint_type == 'wikidata' ? 'Wikidata' : 'DBpedia'}"
				required
			/>
			<div
				use:click_outside
				on:click_outside={() => (show_suggestsions = false)}
				class="{show_suggestsions && suggestions.length > 0
					? 'opacity-100 mt-2'
					: 'opacity-0'} flex flex-col border shadow-lg rounded-lg dark:border-dark-muted dark:bg-dark-bg bg-white z-50 w-full divide-y overflow-hidden transition-all duration-200 h-0"
				style={`height: ${
					show_suggestsions
						? Math.ceil(suggestions.reduce((sum, s) => sum + (s.height ? s.height : 0), 0)) +
						  (suggestions.length - 1) * 1.5
						: 0
				}px`}
			>
				{#if suggestions.length > 0}
					{#each suggestions as suggestion}
						<button
							bind:this={suggestion.button}
							on:click={() => {
								show_suggestsions = false;
								fetch_url(suggestion.uri).then(() => {
									add_nodes();
								});
								close();
							}}
							bind:offsetHeight={suggestion.height}
							class="focus:bg-black/5 dark:focus:bg-black/30 p-2 flex justify-center border-gray-200 dark:border-gray-700 hover:bg-black/5 dark:hover:bg-black/30 !outline-none min-h-[65px]"
						>
							<div class="flex flex-col items-start">
								<span class="font-medium">{suggestion.label}</span>
								<span class="text-sm font-light">{suggestion.type}</span>
							</div>
							<button
								on:click={(e) => {
									e.stopPropagation();
									show_suggestsions = false;
									fetch_url(suggestion.uri);
								}}
								class="ml-auto my-auto mr-2  p-1 rounded-lg bg-transparent hover:bg-black/5 dark:hover:bg-black/30'
								 transition-all duration-200 ease-in-out"
							>
								<Icon src={Plus} class="w-5 h-5" />
							</button>
						</button>
					{/each}
				{:else if !loading}
					<div class="flex flex-col border-gray-200 dark:border-gray-700">
						<span>No suggestions</span>
					</div>
				{/if}
			</div>
			<div hidden={!loading} class="absolute right-2 top-[1.35rem] -translate-y-1/2 w-6 h-6">
				<LoadingCircle />
			</div>
			{#if error && error.length > 0}
				<p class="mt-2 text-sm text-error dark:text-error-dark">
					<span class="font-medium">{error}</span>
				</p>
			{/if}
		</div>
		{#if nodes.length > 0}
			<div class="flex flex-col gap-2" style="margin-right: {scrollbar_width}px;">
				<div class="flex items-center justify-between">
					<h3 class="text-sm font-medium">{nodes.length} Result{nodes.length === 1 ? '' : 's'}</h3>
					<button
						on:click={() => {
							$CurrentGraph.nodes = $CurrentGraph.nodes.filter((n) => !nodes.includes(n));
							nodes = [];
						}}
						class="text-sm text-error dark:text-error-dark"
					>
						Clear
					</button>
				</div>
			</div>
			<div class="space-y-1 max-h-40 overflow-auto">
				{#each nodes as node}
					<div class="flex">
						<div
							class="w-1.5 h-1.5 bg-dark-muted dark:bg-light rounded-full flex-shrink-0 place-self-center mr-2"
						/>
						<div class="truncate flex-grow">
							<a
								href={node.id}
								class="text-sm text-primary leading-6 pb-[2px]"
								target="_blank"
								rel="noreferrer">{node.label}</a
							>
						</div>

						<button class="ml-2 align-middle" on:click={() => remove_node(node)}>
							<Icon src={XMark} class="w-5 h-5" />
						</button>
					</div>
				{/each}
			</div>
		{/if}
	</div>
	<Hr divClass="mb-2" />
	<div class="flex gap-2 items-center">
		<Button on:click={add_nodes} disabled={loading || nodes.length === 0} size="sm">Add</Button>
		<button on:click={close}>Cancel</button>
	</div>
</div>
