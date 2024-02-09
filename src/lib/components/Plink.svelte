<script lang="ts">
	import type { Cursor } from '$lib/Cursor';
	import { flip } from 'svelte/animate';
	import { onMount } from 'svelte';

	let cursors: Cursor[] = [];

	let ourId = '';

	const addCursor = (id: string) => {
		cursors.push({
			id,
			y: 0.5,
			isPressed: false
		});
	};

	const removeCursor = (id: string) => {
		cursors = cursors.filter((c) => c.id != id);
	};

	const moveCursor = (id: string, y: number) => {
		const cursor = cursors.find((c) => c.id === id);
		if (!cursor) {
			console.error(`Could not find cursor with id ${id}`);
			return;
		}
		cursor.y = y;
	};

	const setInitialState = (data: any) => {
		console.log('asdassd', data);
		ourId = data.id;
		cursors = data.users;
	};
	const pressCursor = (id: string, y: number, isPressed: boolean) => {
		let cursor = cursors.find((c) => c.id === id);
		if (!cursor) {
			return;
		}
		cursor.isPressed = isPressed;
		cursors = cursors;
	};

	let socket: WebSocket;
	onMount(() => {
		socket = new WebSocket('ws://192.168.202.8:8000/ws');
		socket.addEventListener('message', (event) => {
			const data = JSON.parse(event.data);
			console.log('Message from server ', data);
			const key = Object.keys(data)[0];
			const value = Object.values(data)[0];
			switch (key) {
				case 'InitialState':
					setInitialState(value);
					break;
				case 'ClientConnect':
					addCursor(value.id);
					break;
				case 'CursorPress':
					pressCursor(value.id, value.y, true);
					break;
				case 'CursorRelease':
					pressCursor(value.id, value.y, false);
					break;
				case 'ClientDisconnect':
					removeCursor(value.id);
					break;
				case 'CursorMove':
					moveCursor(value.id, value.y);
					break;
			}

			cursors = cursors;
		});
	});

	// Listen for messages

	let lastSend = new Date();

	const onMouseMove = (e) => {
		const now = new Date();

		const delta = now.getTime() - lastSend.getTime();
		if (delta < 20) {
			return;
		}

		lastSend = now;

		const normalized = e.clientY / window.innerHeight;

		socket.send(JSON.stringify({ type: 'CursorMove', id: ourId, y: normalized }));
	};

	const onMousePress = () => {
		socket.send(
			JSON.stringify(
				// TODO(@Isha): asd
				{ type: 'CursorPress', id: ourId, y: 0.5 }
			)
		);
	};
	const onMouseRelease = () => {
		socket.send(
			JSON.stringify(
				// TODO(@Isha): aasd
				{ type: 'CursorRelease', id: ourId, y: 0.5 }
			)
		);
	};
</script>

<div class="h-screen flex flex-col border border-black p-2">
	<div class="flex flex-row justify-between">
		<span class="">Your id: {ourId}</span>
		<span>{cursors.length} players online</span>
	</div>
	<div
		class="h-full w-full border border-black p-2 overflow-y-auto"
		on:mousemove={onMouseMove}
		on:mousedown={onMousePress}
		on:mouseup={onMouseRelease}
	>
		<div class="flex flex-col space-y-4 h-full">
			{#each cursors as cursor, index (cursor)}
				<span class="absolute" animate:flip={{ duration: 100 }} style="top: {800 * cursor.y}px"
					>{cursor.id}: {cursor.y} - {cursor.isPressed}</span
				>
			{/each}
		</div>
	</div>
</div>
