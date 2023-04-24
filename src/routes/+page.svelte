<script lang="ts">
	import Aside from '$lib/components/glue/Aside.svelte';
	import Main from '$lib/components/glue/Main.svelte';
	import PageContainer from '$lib/components/glue/PageContainer.svelte';
	import TextInput from '$lib/components/glue/TextInput.svelte';
	import axios from 'axios';

	let query = '';
	let books: string[] = [];
	let selectedBook: string = '';
	let chapters: string[] = [];
	let isLoadingBooks = false;
	let isLoadingSummary = false;
	let loadingChapterTargets: string[] = [];
	const chapterToDetails: { [chapter: string]: string } = {};

	const parseBooksRec = (rec: string) => {
		const regex = /\d+\.\s/g;
		const items = rec.split(regex);
		items.shift();
		return items;
	};

	const parseBookSummary = (summary: string): string[] => {
		return summary?.split(/\n(?=Chapter \d+:)/);
	};

	const fetchBooks = async () => {
		isLoadingBooks = true;
		const { data } = await axios.post('/api/chat', {
			messages: [
				{
					role: 'user',
					content: `list book recommendations about ${query}`
				}
			]
		});
		books = parseBooksRec(data?.response?.content);
		isLoadingBooks = false;
	};

	const fetchBookSummary = async (book: string) => {
		isLoadingSummary = true;
		const { data } = await axios.post('/api/chat', {
			messages: [
				{
					role: 'user',
					content: `short summary of ${book} organized by chapter`
				}
			]
		});
		chapters = parseBookSummary(data?.response?.content);
		isLoadingSummary = false;
	};

	const fetchChapterDetails = async (chapter: string) => {
		loadingChapterTargets = [...loadingChapterTargets, chapter];
		const { data } = await axios.post('/api/chat', {
			messages: [
				{
					role: 'user',
					content: `detailed summary of ${chapter} of book ${selectedBook}`
				}
			]
		});
		chapterToDetails[chapter] = data?.response?.content;
		loadingChapterTargets = loadingChapterTargets?.filter((c) => c !== chapter);
	};
</script>

<PageContainer title="Home" limitWidth={true} layout="aside-main">
	<Aside>
		<form on:submit={fetchBooks}>
			<TextInput class="rounded-full" placeholder="Search for books" bind:value={query} />
		</form>
		<div class="mt-4">
			{#if isLoadingBooks}
				<div class="mt-4 flex justify-center">
					<div class="radial-progress animate-spin" style="--value:40; --size: 1.5rem;" />
				</div>
			{:else}
				<div class="">
					{#each books as book}
						<button
							class="border-b border-base-content/10 p-4 text-left text-sm"
							on:click={() => {
								selectedBook = book;
								fetchBookSummary(book);
							}}>{book}</button
						>
					{/each}
				</div>
			{/if}
		</div>
	</Aside>
	<Main>
		{#if selectedBook}
			<h2 class="text-2xl font-bold leading-tight">{selectedBook}</h2>
			<div class="mt-6">
				{#if isLoadingSummary}
					<div class="flex justify-center">
						<div class="radial-progress animate-spin" style="--value:40; --size: 1.5rem;" />
					</div>
				{:else}
					{#each chapters as chapter}
						<div class="border-b border-base-content/10 py-3">
							<button
								class=" text-left"
								on:click={() => {
									if (!chapterToDetails[chapter]) {
										fetchChapterDetails(chapter);
									}
								}}
							>
								<p>{chapter}</p>
							</button>
							{#if loadingChapterTargets?.includes(chapter)}
								<div class="mt-2 flex justify-center">
									<div class="radial-progress animate-spin" style="--value:40; --size: 1.5rem;" />
								</div>
							{/if}
							{#if chapterToDetails[chapter]}
								<p class="mt-3 whitespace-pre-wrap text-sm leading-normal text-base-content/90">
									{chapterToDetails[chapter]}
								</p>
							{/if}
						</div>
					{/each}
				{/if}
			</div>
		{/if}
	</Main>
</PageContainer>
